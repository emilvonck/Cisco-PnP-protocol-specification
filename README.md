# Cisco PnP protocol specification

This repo contains the Cisco PnP protocol specification and schema files originally published on Cisco DevNet.

It exists because it was surprisingly difficult to find the schemas and protocol documentation when I was developing an internal ZTP service at my previous job.

Hope it helps someone.

Original documentation and archived sources:
- [learn-open-pnp-protocol](https://developer.cisco.com/site/open-plug-n-play/learn/learn-open-pnp-protocol)
- [network-plug-n-play](https://developer.cisco.com/docs/network-plug-n-play/)
- [pnp-schema-files](https://developer.cisco.com/docs/network-plug-n-play/#!schemas/pnp-schema-files)
- [pnp-schema-files (web.archive.org)](https://web.archive.org/web/20230325210558/https://developer.cisco.com/docs/network-plug-n-play/#!schemas/pnp-schema-files)

## PnP service discovery options
### DHCP option 43 with Vendor Class Identifier (option 60)

Avoid using boot options (66, 67, 150, etc.) in the DHCP policy for ZTP devices.

They may conflict with PnP discovery. Use Vendor Class Identifier-based policy filters instead:

```yaml
vendor_class:
- hex: '0x636973636F706E70'
  # case-sensitive
  ascii: 'ciscopnp'
```

The following options __must be defined and empty__:
- Option 66: `none`
- Option 67: `none`
- Option 150: `none`

Option 43 is used to configure the PnP agent using the following format:
```
<prefix><arguments>
```

Prefix values:
- Typecode: __5__ (Cisco PnP service)
- Feature opcode: __Active (A)__ or __Passive (P)__, only active is supported
- Version - PnP version, only __1__ is supported
- Debug - PnP client debug logging, __N__ - off, __D__ - on
- Arguments - PnP configuration parameters separated by ";"

Argument values:
- B - Address type, 1 - DNS host, 2 - IPv4 address, 3 - IPv6 address
- I - FQDN or IP address of PnP service
- K - Transport type, 4 - HTTP, 5 - HTTPS
- J - (optional) Service connection port, defaults to 80 for HTTP and 443 for HTTPS
- N - (optional) Username for the PnP service connection
- O - (optional) Password for the PnP user
- T - (optional) URL of the CA trustpool bundle, required for HTTPS transport
- Z - (optional) NTP server IP address, required for HTTPS transport

Example value:
```
5A1N;B1;Ipnpserver.zone.lab;K5;Thttp://pnpserver.zone.lab/ca/trustpool;Z172.24.172.11
```

__CA trustpool__

When HTTPS transport is used (K5) and CA URL is not defined (T), the PnP agent tries to download the trustpool from the predefined PnP service path `/ca/trustpool`.

The service certificate must contain __both the FQDN and the resolved IP address in the SAN!__

Example:
```
FQDN: pnpserver.lab.internal
IP: 10.252.252.10
Certificate SAN:dns=pnpserver.lab.internal, SAN:ip=10.252.252.10
```

### DNS Discovery
- Networking device is cabled and powered on. Since the startup configuration in NVRAM is empty, PnP agent is triggered and sends “Cisco PnP” in DHCP Option 60 in DHCP DISCOVER message.
- Since DHCP server is not configured to recognize “Cisco PnP” in Option 60, it ignores Option 60. DHCP server assigns an IP address and sends DHCP offer along with configured domain name and DNS server IP address.
- PnP agent reads domain name and formulates fully qualified PnP server hostname by appending domain name to the string “pnpserver”. If the domain name is “xyz.com”, fully qualified hostname of PnP server would be “pnpserver.xyz.com”. PnP agent resolves “pnpserver.xyz.com” for its IP address with the DNS server received in the DHCP options.

### Example DHCP server config
```
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;
subnet 172.24.172.0 netmask 255.255.255.0 {
    range 172.24.172.200 172.24.172.254;
    option subnet-mask 255.255.255.0;
    option routers 172.24.172.1;
    option domain-name-servers 172.24.172.24;
    option domain-name "zone.lab";
    default-lease-time 600000;
    max-lease-time 720000;
    if option vendor-class-identifier = "ciscopnp" {
        option vendor-encapsulated-options "5A1N;B1;Ipnpserver.zone.lab;K5;Thttp://pnpserver.zone.lab/ca/trustpool;Z172.24.172.11";
    }
}
```

### Example DNS zone config
```
$TTL 5m    ; default TTL for zone
$ORIGIN zone.lab. ; base domain-name
@         IN      SOA   ns1.zone.lab. hostmaster.zone.lab. (
                                2003080800 ; serial number
                                12h        ; refresh
                                15m        ; update retry
                                3w         ; expiry
                                5m         ; minimum
                                )
           IN      NS      ns1.zone.lab.
ns1        IN      A       172.24.172.24
pnpserver  IN      A       172.24.172.24
ntp        IN      A       172.24.172.11
```

### Work Request
| | |
| --- | --- |
| __Purpose__ | Work Request message is initiated by the PnP agent to check with the PnP server for any new work requests. PnP agent sends this message after discovering the server or when a PnP profile is configured and also at the beginning of every retry. This is applicable only when the transport protocol is HTTP and HTTPS. |
| __Type__ | info |
| __URL__ | pnp/WORK-REQUEST |
| __HTTP method__ | POST |

Example:
```xml
<pnp xmlns="urn:cisco:pnp" version="1.0" udi="PID:WS-C3750X-24T-E,VID:V04,SN:FDO1703P2EB">
  <info xmlns="urn:cisco:pnp:work-info" correlator="CiscoPnP-1.0-1465-25A1212C">
    <deviceId>
      <udi>PID:WS-C3750X-24T-E,VID:V04,SN:FDO1703P2EB</udi>
      <hostname>Router</hostname>
      <authRequired>true</authRequired>
      <viaProxy>false</viaProxy>
      <securityAdvise>Password in clear text in unsecured transport</securityAdvise>
    </deviceId>
  </info>
</pnp>
```

### Bye
| | |
| --- | --- |
| __Purpose__ | Bye message is sent by the PnP server to acknowledge the receipt of PnP response and also signal the end of the transaction. This is applicable only when the transport protocol is HTTP and HTTPS. |
| __Type__ | info |
| __URL__ | 200 OK for pnp/WORK-RESPONSE |
| __HTTP method__ | POST |

Example:
```xml
<pnp xmlns="urn:cisco:pnp" version="1.0" udi="PID:WS-C3750X-24T-E,VID:V04,SN:FDO1703P2EB">
  <info xmlns="urn:cisco:pnp:work-info" correlator="CiscoPnP-1.0-1465-25A1212C">
    <workInfo>
      <bye/>
    </workInfo>
  </info>
</pnp>
```

## Cisco PnP Schema files

| Service  | Files  | Original URL (no longer available) | Description  |
| --- | --- | --- | --- |
| Backoff  | pnp_backoff_body.xsd  | [pnp_backoff_body.xsd](https://pubhub.devnetcloud.com/pnp_backoff_body.xsd)  | Request the agent to back off from sending more work-request for a period of time.  |
|  | pnp_backoff_svc.xsd  | [pnp_backoff_svc.xsd](https://pubhub.devnetcloud.com/pnp_backoff_svc.xsd)  |  |
| Capability  | pnp_capability_body.xsd  | [pnp_capability_body.xsd](https://pubhub.devnetcloud.com/pnp_capability_body.xsd)  | Request to query what services are supported by the agent.  |
|  | pnp_capability_svc.xsd  | [pnp_capability_svc.xsd](https://pubhub.devnetcloud.com/pnp_capability_svc.xsd)  |  |
| Certificate Install  | pnp_certificate_install_body.xsd | [pnp_certificate_install_body.xsd](https://pubhub.devnetcloud.com/pnp_certificate_install_body.xsd) | Request the agent to install a trustpoint or trustpool certificate into the devices.  |
|  | pnp_certificate_install_svc.xsd  | [pnp_certificate_install_svc.xsd](https://pubhub.devnetcloud.com/pnp_certificate_install_svc.xsd)  |  |
| CLI Config  | pnp_cli_config_body.xsd  | [pnp_cli_config_body.xsd](https://pubhub.devnetcloud.com/pnp_cli_config_body.xsd)  | Request to execute a configuration CLI.  |
|  | pnp_cli_config_svc.xsd  | [pnp_cli_config_svc.xsd](https://pubhub.devnetcloud.com/pnp_cli_config_svc.xsd)  |  |
| CLI Exec  | pnp_cli_exec_body.xsd  | [pnp_cli_exec_body.xsd](https://pubhub.devnetcloud.com/pnp_cli_exec_body.xsd)  | Request to execute a exec level CLI.  |
|  | pnp_cli_exec_svc.xsd  | [pnp_cli_exec_svc.xsd](https://pubhub.devnetcloud.com/pnp_cli_exec_svc.xsd)  |  |
| Config Upgrade  | pnp_config_upgrade_body.xsd  | [pnp_config_upgrade_body.xsd](https://pubhub.devnetcloud.com/pnp_config_upgrade_body.xsd)  | Upgrade the device’s configuration to a new config.  |
|  | pnp_config_upgrade_svc.xsd  | [pnp_config_upgrade_svc.xsd](https://pubhub.devnetcloud.com/pnp_config_upgrade_svc.xsd)  |  |
| Device Authentication | pnp_device_auth_body.xsd  | [pnp_device_auth_body.xsd](https://pubhub.devnetcloud.com/pnp_device_auth_body.xsd)  | Request to authenticate the device.  |
|  | pnp_device_auth_svc.xsd  | [pnp_device_auth_svc.xsd](https://pubhub.devnetcloud.com/pnp_device_auth_svc.xsd)  |  |
| Device Info  | pnp_device_info_body.xsd  | [pnp_device_info_body.xsd](https://pubhub.devnetcloud.com/pnp_device_info_body.xsd)  | Request to obtain device’s profile which includes device UDI, filesytem, hardware info and etc. |
|  | pnp_device_info_svc.xsd  | [pnp_device_info_svc.xsd](https://pubhub.devnetcloud.com/pnp_device_info_svc.xsd)  |  |
| File Transfer  | pnp_file_transfer_body.xsd  | [pnp_file_transfer_body.xsd](https://pubhub.devnetcloud.com/pnp_file_transfer_body.xsd)  | Request to copy a file.  |
|  | pnp_file_transfer_svc.xsd  | [pnp_file_transfer_svc.xsd](https://pubhub.devnetcloud.com/pnp_file_transfer_svc.xsd)  |  |
| Image Install  | pnp_image_install_body.xsd  | [pnp_image_install_body.xsd](https://pubhub.devnetcloud.com/pnp_image_install_body.xsd)  | Request the agent to install a new image.  |
|  | pnp_image_install_svc.xsd  | [pnp_image_install_svc.xsd](https://pubhub.devnetcloud.com/pnp_image_install_svc.xsd)  |  |
| Licensing  | pnp_licensing_body.xsd  | [pnp_licensing_body.xsd](https://pubhub.devnetcloud.com/pnp_licensing_body.xsd)  | Request to install and configure new license.  |
|  | pnp_licensing_svc.xsd  | [pnp_licensing_svc.xsd](https://pubhub.devnetcloud.com/pnp_licensing_svc.xsd)  |  |
| Redirection  | pnp_redirection_body.xsd  | [pnp_redirection_body.xsd](https://pubhub.devnetcloud.com/pnp_redirection_body.xsd)  | Request to redirect PnP Profile to use another PnP server.  |
|  | pnp_redirection_svc.xsd  | [pnp_redirection_svc.xsd](https://pubhub.devnetcloud.com/pnp_redirection_svc.xsd)  |  |
| Reload  | pnp_reload_body.xsd  | [pnp_reload_body.xsd](https://pubhub.devnetcloud.com/pnp_reload_body.xsd)  | Request the agent to reload.  |
|  | pnp_reload_svc.xsd  | [pnp_reload_svc.xsd](https://pubhub.devnetcloud.com/pnp_reload_svc.xsd)  |  |
| Script  | pnp_script_body.xsd  | [pnp_script_body.xsd](https://pubhub.devnetcloud.com/pnp_script_body.xsd)  | Request to execute a script.  |
|  | pnp_script_svc.xsd  | [pnp_script_svc.xsd](https://pubhub.devnetcloud.com/pnp_script_svc.xsd)  |  |
| SMU  | pnp_smu_body.xsd  | [pnp_smu_body.xsd](https://pubhub.devnetcloud.com/pnp_smu_body.xsd)  | Request to install a SMU.  |
|  | pnp_smu_svc.xsd  | [pnp_smu_svc.xsd](https://pubhub.devnetcloud.com/pnp_smu_svc.xsd)  |  |
| Topology  | pnp_topology_body.xsd  | [pnp_topology_body.xsd](https://pubhub.devnetcloud.com/pnp_topology_body.xsd)  | Request topology of the device.  |
|  | pnp_topology_svc.xsd  | [pnp_topology_svc.xsd](https://pubhub.devnetcloud.com/pnp_topology_svc.xsd)  |  |
| Work Info  | pnp_work_info_body.xsd  | [pnp_work_info_body.xsd](https://pubhub.devnetcloud.com/pnp_work_info_body.xsd)  | Start the request.  |
|  | pnp_work_info_svc.xsd  | [pnp_work_info_svc.xsd](https://pubhub.devnetcloud.com/pnp_work_info_svc.xsd)  |  |
