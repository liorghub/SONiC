# In-Band ZTP HLD

# High Level Design Document

#### Rev 0.1

# Table of Contents
- [Revision](#revision)
- [Scope](#scope)
- [Definitions/Abbreviation](#definitionsabbreviation)
- [1 Overview](#1-overview)
- [2 Requirements](#2-requirements)
  - [2.1 Functional requirements](#21-functional-requirements)
  - [2.2 Configuration and Management Requirements](#22-configuration-and-management-requirements)
- [3 Modules design](#3-modules-design)
  - [3.1 ZTP provision over in-band network on init](#31-ZTP-provision-over-in-band-network-on-init)
    - [3.1.1 config-setup service](#311-config-setup-service)
    - [3.1.2 interfaces-config service](#312-interfaces-config-service)
    - [3.1.3 ZTP service](#313-ztp-service)
    - [3.1.4 SWSS COPP](#314-swss-copp)
  - [3.2 ZTP provision over in-band network on runtime](#32-ZTP-provision-over-in-band-network-on-runtime)
  - [3.3 Config DB](#33-config-db)
  - [3.4 CLI commands](#34-cli-commands)


# Revision
| Rev | Date     | Author          | Change Description                 |
|:---:|:--------:|:---------------:|------------------------------------|
| 0.1 | 14/07/22 | Lior Avramov    | Initial version                    |

# Scope
This document describes the high level design of the in-band ZTP feature in SONiC.

# Definitions/Abbreviation
| Abbreviation  | Description                               |
|---------------|-------------------------------------------|
| ZTP           | Zero-touch provisioning                   |
| DHCP          | Dynamic host configuration protocol       |


# 1 Overview
## 1.1 ZTP overview
When a newly deployed SONiC switch boots for the first time, it should allow automatic configuration of the switch without user intervention. This framework is called ZTP.
ZTP allows switch that boots from factory default to communicate with remote provisioning server (DHCP server), download a file called ZTP json and perform the configuration tasks listed in it. 

Configuration tasks are defined with the corresponding plugin to be applied by ZTP service. Plugin can be config_db.json to apply, SW image to install or SNMP community string. ZTP allows to perform one or more configuration tasks. It also allow ordering of those tasks as defined in the ZTP json. 

DHCP option 67 (59 for DHCPv6) in the DHCP offer contains the url to the ZTP json. ZTP service will download the ZTP json, process it and execute the configuration tasks listed in it. Alternitively, ZTP can download a simple provisioning script and execute it. DHCP option 239 (239 for DHCPv6) in the DHCP offer contains the url to the script. 
ZTP can also download minigraph xml or ACL json used by updategraph service. DHCP option 225 and 226 in the DHCP offer contains the url for those. 

The following CLI commands are used to manage ZTP service.
| Command         | Description                              |
|-----------------|------------------------------------------|
| ztp enable      | enable ZTP service                       |
| ztp disable     | stop and disable ZTP service             |
| ztp run         | manually restart a new ZTP session.      |
| show ztp status | displays the current ZTP configuration   |

## 1.2 In-band ZTP overview
When using in-band ZTP switch will communicate with remote provisioning server using the high-speed interfaces allowing the user to provision the switch using the in-band network in adition to the out-of-band network.

# 2 Requirements

## 2.1 Functional requirements
In-band ZTP should meet the following requirements:
- ZTP should be able to provision the switch over in-band network in addition to the out-of-band network.
- DHCP discovery should be performed on all in-band interfaces.
- The first interface to provide provisioning data will be used and any provisioning data provided by other interfaces is ignored.

## 2.2 Configuration and management requirements
- In-band ZTP is enabled by default when the ZTP package is included.
- User can disable in-band ZTP by adding "feat-inband" : false to /host/ztp/ztp_cfg.json.
- In-band ZTP requires registration of 2 new traps, SAI_HOSTIF_TRAP_TYPE_DHCP_L2 and SAI_HOSTIF_TRAP_TYPE_DHCPV6_L2 traps. Traps should be enabled when ZTP starts and disabled when it finishes.

# 3 Modules design

![In-band ZTP modules](images/modules-design.svg)

Service config-setup creates ZTP configuration, interface-config is in charge of DHCP, SWSS registers the required traps and then ZTP is doing the work.
More details on each service in the following chapters.

## 3.1 ZTP provision over in-band network on init

### 3.1.1 config-setup service
During init, systemd runs config-setup service that performs the following:

![config-setup](images/config-setup.svg)

- Check whether config_db.json exists, if yes, nothing to do in terms of ZTP, exit.
- Else, create the following files using sonic-cfggen:
1. config_db.json (using ztp-config.j2) with 4 tables: DEVICE_METADATA, ZTP, PORT and COPP_TRAP.
* DEVICE_METADATA table data (product name, serial number) are being read using decode-syseeprom command.
* PORT table data (alias, lanes, speed, admin status and mtu) are being read from platfrom.json according to the hwsku (if in-band ZTP is disabled, ports admin state will be set to down).
* ZTP table data (admin state of in-band ZTP, ipv4 ZTP and ipv6 ZTP) are being read from file called defaults.py, this file holds all ZTP defines (sub features admin state, helper files location, etc.).
* COPP_TRAP table data are the DHCP traps required for in-band ZTP. Those are SAI_HOSTIF_TRAP_TYPE_DHCP_L2 and SAI_HOSTIF_TRAP_TYPE_DHCPV6_L2 (more details at [3.1.4 SWSS COPP](#314-swss-copp))
2. /etc/network/ifupdown2/policy.d/ztp_dhcp.json (using ifupdown2_dhcp_policy.j2): this file contains DHCPv6 unique identifier type for each interface, we use LL (link-layer address).

- Run config reload to load the newly created config_db.json, and if ZTP is enabled, stop the service and delete ZTP json to prepare for a new session.

### 3.1.2 interfaces-config service
After config reload, during init, systemd runs service interfaces-config. It perfrom the following:

![interfaces-config](images/interfaces-config.svg)

Check if file ztp_dhcp.json exist, if yes:
- Read interfaces data from PORT_TABLE in app DB (admin_status, alias, lanes, mtu, oper_status and speed)
- Use sonic-cfggen to create the following files (use interface data as data source):
1. /etc/network/interfaces file (using interfaces.j2). This file contains the interfaces network configuration like IP address, network netmask, DHCP enable etc.
2. /etc/dhcp/dhclient.conf file (using dhclient.conf.j2). This file defines the DHCP information provided to the client by the server (DHCP ZTP options and requests).
3. /etc/sysctl.d/90-dhcp6-systcl.conf (using 90-dhcp6-systcl.conf.j2). This file contains DHCPv6 related configuration for the mgmt. interface only, accept_ra (accept router advertisements) and accept_ra_defrtr (learn default router in router advertisement).
- Restart networking service, this will start DHCP discovery on the managment interface only (at this phase, SWSS did not create the interfaces yet).

If ztp_dhcp.json does not exist, it means ZTP is disabled, so create same files mentioned above but without the request for DHCP ZTP options.

### 3.1.3 ZTP service
Systemd runs ZTP service after interfaces-config. ZTP performs the following:

![ztp-engine](images/ztp-engine.svg)

- Run discovery, in discovery method we check if we got one of the DHCP options from DHCP server.
- In the first call to discovery we expect nothing since we didnt kickstart DHCP on the inband interfaces yet.
- Wait for in-band interfaces to be created (by reading interfaces names from config DB and poll for existance of /sys/class/net/${PORT}, we do it with timeout of 120 seconds.
- Read oper state of all in-band interfaces (read PORT_TABLE in app DB) and if there is a link up change, perform restart to interfaces-config service, this will start DHCP discovery on all interfaces.
- DHCP process starts.
- DHCP hook /etc/dhcp/dhclient-enter-hooks.d/inband-ztp-ip sets the offered IP address on the in-band interface.
- DHCP hook /etc/dhcp/dhclient-exit-hooks.d/ztp parses the option and writes it to a file in the filesystem.
- Run discovery again, this time we expect that one of the options was parsed, we read the url and download the ZTP json. This is the the order of precedence:
1. ZTP json file exist as part of the SW image.
2. Download ZTP json specified in DHCP Option-67.
3. Download ZTP json specified in DHCPv6 Option-59.
4. Download simple provisioning script specified in DHCP Option-239.
5. Download simple provisioning script specified in DHCPv6 Option-239.
6. Download Minigraph xml and ACL json specified in DHCP Options 225 and 226.

- Process the configuration sections appear in ZTP json (/host/ztp/ztp_data.json). Each section contains plugin to execute. There are several types of plugins:
1. configdb-json - the plugin is used to download config DB json file and apply it. A config reload is performed during which various SWSS services may restart.
2. firmware - this plugin is used for image management on a switch. It can be used to install, remove and boot selection of images. sonic_installer utility is used by this plugin to perform the supported functions.
3. connectivity-check - this plugin is used to ping a remote host and verify if the switch is able to reach the remote host.
4. snmp - this plugin is used to configure SNMP community string on.
5. script - this plugin is used to run a user script.

In the below example, there are 3 sections to process:
```
{
	"ztp": {
		"01-configdb-json": {
			"url": {
				"source": "http://192.168.0.1/ZTP/config_db.json",
				"destination": "/etc/sonic/config_db.json"
			}
		},
		"02-configdb-json": {
			"url": {
				"source": "http://192.168.0.1/ZTP/config_db-FEATURE.json",
				"destination": "/etc/sonic/config_db.json"
			},
			"dynamic-url": {
				"source": {
					"prefix": "http://192.168.0.1/ZTP/",
					"identifier": "hostname",
					"suffix": "_config_db.json"
				},
				"destination": "/etc/sonic/config_db.json"
			}
		},
		"03-script": {
			"plugin": {
				"url": {
					"source": "http://192.168.0.1/ZTP/03-script.sh"
				}
			},
			"reboot-on-failure": "true"
		}
	}
}
```
- Last step is to clear ZTP configuration from the switch:
1. Clear ZTP and COPP tables from config DB using sonic-db-cli.
2. Clear ZTP and COPP tables from config_db.json.
3. Delete file ztp_dhcp.json and restart interfaces-config service. When ztp_dhcp.json does not exist, interfaces-config service will create dhclient.conf without request of ZTP options.

### 3.1.4 SWSS COPP
For making DHCP packets arrive to CPU, need to register the following traps:
- SAI_HOSTIF_TRAP_TYPE_DHCP_L2
- SAI_HOSTIF_TRAP_TYPE_DHCPV6_L2

Service config-setup creates new config_db.json during init and performs config relaod to load it. The following will be added to this config_db.json:
```
COPP_TRAP|l2dhcp
  always_enabled:true
```
ZTP engine will delete this key from config DB when ZTP finishes its work.

COPP manager listens to changes in COPP_TRAP table, therefore will handle and register the traps.
When COPP manger handles changes in COPP_TRAP table it uses data being read in init and kept in the cache, this data is generated from copp_cfg.j2.

Following will be added to copp_cfg.j2:
```
"l2dhcp": {
  "trap_ids": "l2dhcp,l2dhcpv6",
  "trap_group": "queue4_group4"
},
```
```
"queue4_group5": {
  "trap_action":"copy",
  "trap_priority":"4",
  "queue": "4",
  "meter_type":"packets",
  "mode":"sr_tcm",
  "cir":"600",
  "cbs":"600",
  "red_action":"drop"
}
```
And extend trap_id_map with:
```
static map<string, sai_hostif_trap_type_t> trap_id_map = {
  {"l2dhcp", SAI_HOSTIF_TRAP_TYPE_DHCP_L2},
  {"l2dhcpv6", SAI_HOSTIF_TRAP_TYPE_DHCPV6_L2}
};
```

## 3.2 ZTP provision over in-band network on runtime
User can force ZTP service to run using CLI command "ztp run", this will restart ZTP operation.
In this scenario, ztp-engine performs the following:
1. Verify all ZTP sub features (inband, ipv4, ipv6) are enabled in config DB. If not, create config_db.json and ztp_dhcp.json (same as config-setup does on init) and perfrom config reload to load the newly created config_db.json.
2. Rest of the flow is as described in [3.1.3 ZTP service](#313-ztp-service)

## 3.3 Config DB
When ZTP is enabled, the following will be added to config DB at init and will be deleted once ZTP will finish its operation.
```
"ZTP" : {
  "mode" : {
    "profile" : "active",
    "inband" : "true",
    "out-of-band" : "true",
    "ipv4" : "true",
    "ipv6" : "true",
    "product-name" : "",
    "serial-no" : ""
  }
}
```
## 3.4 CLI commands
CLI commands definition and examples can be found in ZTP HLD https://github.com/sonic-net/SONiC/blob/master/doc/ztp/ztp.md#6-configuring-ztp
