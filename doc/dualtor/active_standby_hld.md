<!-- vscode-markdown-toc -->
* [Revision](#Revision)
* [Scope](#Scope)
* [Content](#Content)
* [1 Requirement](#Requirement)
* [2 Theary of Operation](#ThearyofOperation)
	* [2.1 Smart Y-Cable (MUX Cable) Behavior](#SmartY-CableMUXCableBehavior)
	* [Routing Behavior](#RoutingBehavior)
* [3 Data Schema](#DataSchema)
	* [3.1 Config DB](#ConfigDB)
	* [3.2 APP DB](#APPDB)
	* [3.3 STATE DB](#STATEDB)
* [4 Command Line](#CommandLine)
	* [4.1 MUX Configuration](#MUXConfiguration)
		* [4.1.1 Configuration Example](#ConfigurationExample)
	* [4.2 MUX Status](#MUXStatus)
		* [4.2.1 MUX Configuration Status](#MUXConfigurationStatus)
		* [4.2.2 MUX Operation Status](#MUXOperationStatus)
	* [4.3 MUX cableinfo](#MUXcableinfo)
		* [4.3.1 Example](#Example)
	* [4.4	Cable Firmware Commands](#CableFirmwareCommands)
		* [4.4.1	Muxcable Firmware Version](#MuxcableFirmwareVersion)
		* [4.4.2	Muxcable Firmware Download](#MuxcableFirmwareDownload)
		* [4.4.3	Activate Firmware](#ActivateFirmware)
		* [4.4.5 Rollback Firmware](#RollbackFirmware)

<!-- vscode-markdown-toc-config
	numbering=false
	autoSave=true
	/vscode-markdown-toc-config -->
<!-- /vscode-markdown-toc -->
# SONiC Active-Standby Dual ToR HLD

## <a name='Revision'></a>Revision

|  Rev  |   Date   |    Author     | Change Description             |
| :---: | :------: | :-----------: | ------------------------------ |
|  0.1  | 01/03/23 |  Jing Zhang   | Initial version                |

## <a name='Scope'></a>Scope

This document provides an introduction of the high-level design of SONiC Dual ToR with Smart Y-Cable solution. 

## <a name='Content'></a>Content


## <a name='Requirement'></a>1 Requirement

Switch traffic to a healthy link / ToR when there is a link failure. 

## <a name='ThearyofOperation'></a>2 Theary of Operation 
### <a name='SmartY-CableMUXCableBehavior'></a>2.1 Smart Y-Cable (MUX Cable) Behavior 
* From ToR to NIC both links will be up, but only active link traffic will be forwarded to the NIC side. 
* From NIC to ToR, traffic will be broadcasted to both ToR. 
* There will be no link down during mux switchovers. 
* Expect a few packets corrupted/dropped from ToR to NIC during the mux switchover.
* Expect no traffic disruption during mux switchover from NIC to ToR. 

### <a name='RoutingBehavior'></a>Routing Behavior
* Both ToR will have same VLAN configuration as well as virtual mac address. 
* Both ToR will advertise VLAN prefix to T1
* One of the ToR port will be active state, the other ToR port will be standby state. 
* For southbound traffic (from T1 to server), active port will forward it down directly to server, standby port will forward it to the peer ToR (through IPinIP tunnel). 
* For northbound traffic (from server to T1), active port forward it directly. ToR will drop all the traffic received on the standby port. 

## <a name='DataSchema'></a>3 Data Schema 
### <a name='ConfigDB'></a>3.1 Config DB 
* MUX_LINKMGR|LINK_PROBE
  * interval_v4: 100  ; heartbeat probe interval in millisecond, default value is 100.   
  * interval_v6: 1000 ; heartbeat probe interval for ipv6 in millisecond, default value is 1000.  
  * positive_signal_count: 1 ; event count to confirm a positive state transition, i.e. _unknown_ to _active_.    
  * negative_signal_count: 3 ; event count to confirm a negative state transtion, i.e. _active_ to _unknown_.  
  * suspend_timer: 500 ; heartbeats will be suspended for duration of suspend_timer when heartbeat state transits from _active_ to _unknown_.  
    \*\*This field is not in use anymore. suspending timeout will depend on interval_v4 and negative_signal_count.  
  * interval_pck_loss_count_update: 300 ; interval to report heartbeat loss count to stream telemetry in heartbeat counts, minimum value is 50, default value is 300.  
* MUX_LINKMGR|MUXLOGGER
  * log_verbosity: trace|debug|info|warning|error|fatal ; log verbosity of _linkmgrd_.  
* MUX_CABLE|<PORTNAME>
  * server_ipv4: ipv4 prefix 
  * server_ipv6: ipv6 prefix
  * state: auto|manual|active|standby ; `auto` mode port will switch mux state proactively (based on heartbeat loss or link down event etc.), `manual` mode will switch reactively to any mux state change, `active` or `standby` will switch mux first and act like `manual` mode. 
* PEER_SWITCH|<DEVICENAME>
  * address_ipv4: ipv4 address
* TUNNEL|<MUXTUNNELNAME>
  * tunnel_type: IPINIP
  * dst_ip: IPv4 address
  * dscp_mode: uniform
  * encap_ecn_mode: standard
  * ecn_mode: copy_from_outer
  * ttl_mode: pipe
* DEVICE_METADATA|localhost
  * type: ToRRouter
  * peer_switch: hostname of peer switch
  * subtype: DualToR
* VLAN|<VlanName>
  * mac: mac address

### <a name='APPDB'></a>3.2 APP DB

* MUX_CABLE_TABLE:<PORTNAME>
  * state: active|standby ; for linkmgrd to communicate with orchagent.
* HW_MUX_CABLE_TABLE:<PORTNAME>
  * state: active|standby ; for orchagent to communicate with transceiver daemon.
* MUX_CABLE_COMMAND_TABLE:<PORTNAME>
  * command: probe ; for linkmgrd to communicate with transceiver daemon.
* MUX_CABLE_RESPONSE_TABLE:<PORTNAME>
  * response: active|standby|unknown

### <a name='STATEDB'></a>3.3 STATE DB

* MUX_CABLE_TABLE|<PORTNAME>
  * state: active|standby|unknown|error ; written by orchagent.
* HW_MUX_CABLE_TABLE|<PORTNAME>
  * state: active|standby|unknown ; written by transceiver daemon.
* MUX_LINKMGR_TABLE|<PORTNAME>
  * state: healthy|unhealthy|uninitialized ; written by linkmgrd to reflect current healthy state combining mux state and link prober state. Uninitialized indicates that this port did not complete initialization and is considered incapable for auto failover. 
* MUX_METRICS_TABLE|<PORTNAME>
  * <app name>_switch_<target state>_start: yyyy-mmm-dd hh:mm:ss.uuuuu ; time when \<app name\> started switch operation to state \<target state\>. \<app name\> is one of transceiver daemon, orchagent, or linkmgrd and \<target state\> is one of active, standby, or unknown.
  * <app name>_switch_<target state>_end: yyyy-mmm-dd hh:mm:ss.uuuuuu ; time when \<app name\> completed switch operation to state \<target state\>. \<app name\> and \<target state\> are as defined above.

## <a name='CommandLine'></a>4 Command Line
### <a name='MUXConfiguration'></a>4.1 MUX Configuration 
Configuration data will be updated via cli module `config`. MUX operation mode could be set using the folloing command:
```
config muxcable mode {active} {<portname>|all} [--json]
```
The above command will set the MUX operation mode for specific port `portname` if provided. If `all` keyword is provided, the operation mode will apply to all ports. 
#### <a name='ConfigurationExample'></a>4.1.1 Configuration Example
```
ex: config muxcable mode active Ethernet4 [--json]

output case 1:
RC: 0
{
  "Ethernet4": "OK"
}

output case 2:
RC: 1

output case 3：
RC: 0
{
  "Ethernet4": "INPROGRESS"
}
```
In case 1, `OK` output would be returned ifthe mux is in active state already. Return code of 1 indicates failure for case 2. Case 3 of `INPROGRESS` for mux port in healthy standby. 

Return code 0 indicates configuring is successful, all other RCs will be considered failed. 

##### 4.1.1.1 Cli Transition Table for Config Mode 
Current STATE in STATE_DB MUX_CABLE_TABLE | Current MODE in CONFIG_DB MUX_CABLE | Cli command MODE (per port) | Transition to (Cli interpretor)
:----------------------------------------:|:-----------------------------------:|:---------------------------:|:------------------------------:
Active | Auto | Auto | Do nothing. Display OK.
Active | Auto/Manual | Active | Write `auto` to CONFIG_DB. Display OK.
Active | Active | Auto | Write `auto` to CONFIG_DB. Display OK.
Active | Active | Active | Do Nothing. Display OK. 

### <a name='MUXStatus'></a>4.2 MUX Status
MUX status will be retrieved using `show` commands. 

#### <a name='MUXConfigurationStatus'></a>4.2.1 MUX Configuration Status

To view MUX configuration data, `show` module wil be extended to support the following command: 
```
show muxcable config [portname] [--json]
```
##### 4.2.1.1 Example
```
ex: show muxcable config --json
output:
{
   “MUX_CABLE”: {
       “PEER_TOR”: “10.10.10.3”,
       “LINK_PROBER”: {
           “INTERVAL”: {
               “IPv4”: 100,
               “IPv6”: 1000,
           },
           “TIMEOUT”: 3,
       },
       “PORTS”: {
           “ETHERNET0”: {
               “STATE”: “ACTIVE”,
               “SERVER”: {
                   “IPv4”: “192.168.1.10”,
                   “IPv6”: “fc00::75”,
               }
           },
           “ETHERNET4”: {
               “STATE”: “AUTO”,
               “SERVER”: {
                   “IPv4”: “192.168.1.40”,
                   “IPv6”: “fc00::ab”,
               }
           }
       }
    }
}
```
Return Code 0 means command is executed successfully, all other RCs are to be considered failed.

#### <a name='MUXOperationStatus'></a>4.2.2 MUX Operation Status

Mux status can be active/standby/unknown. 

The reason to incorporate an unknown state is that there could be a case when cable is not powered on or cable is disconnected, faulty cable etc. In those scenarios the ToR cannot access the status of the mux direction though i2c. So, the ToR would reflect the status as unknown in these scenarios.

To view the current operation status of the MUX cable, show module will support the following command:
```
show muxcable status [portname] [--json]
```

##### 4.2.2.1 Single Port Example 

```
show muxcable status Ethernet0 –-json 
output:
{
   “MUX_CABLE”: {
       “ETHERNET0”: {
           “STATUS”: “ACTIVE”,
	“HEALTH”: “HEALTHY”
        }
    }
}
```

##### 4.2.2.2 All Ports Example

```
show muxcable status --json
output:
{
   “MUX_CABLE”: {
       “ETHERNET0”: {
           “STATUS”: “ACTIVE”,
	“HEALTH”: “HEALTHY”
        },
       “ETHERNET4”: {
           “STATUS”: “STANDBY”,
	“HEALTH”: “HEALTHY”
        },
       .
       .
       “ETHERNET<N>”: {
           “STATUS”: “STANDBY”,
	“HEALTH”: “UNHEALTHY”
        }
    }
}
``` 

Return Code 0 means show status is successful , everything else is considered failed.

### <a name='MUXcableinfo'></a>4.3 MUX cableinfo 

To view the cable vendor information, command below can be used. 
```
show muxcable cableinfo <PORTNAME>
```

#### <a name='Example'></a>4.3.1 Example 

This command can be utilized to get unique part number and cable vendor name. 
```
show muxcable cableinfo Ethernet0
Output: 
Vendor                   Model
---------------          -------------
4.4	Credo                CACL1X321P2PA1M  
```

RC 0 will mean success all other RCs will mean error. 


### <a name='CableFirmwareCommands'></a>4.4	Cable Firmware Commands 

A group of muxcable firmware commands will be supported for cable firmware downloading, upgrading and checking. 

The way to upgrade the firmware would be hwproxy will need to first download the firmware, followed by activating the firmware as described in the examples. By default the non active bank is picked up on all vendor cables for downloading and activating the cable firmware. The RC of both the commands will determine if the upgrade process was success or not, followed by a check of the firmware version using the show firmware version. As usual RC 0 indicates a cli execution was successful and any other return code indicates it is unsuccessful. 

Update: There is now a show muxcable download_firmware status <port> available which can be used after download command to check whether the download is in progress or not.  For activate and rollback we have to rely on the RC for validating a successful execution of CLI. Show muxcable firmware version can also be used to post validate whether the firmware version activate/rollback was successful or not. 

#### <a name='MuxcableFirmwareVersion'></a>4.4.1	Muxcable Firmware Version
```
Show muxcable firmware version <port>
```

Example: 
``` 
show muxcable firmware version Ethernet52 
Output: 
{ 
    "version_self_active": 0.7MS, 
    "version_self_inactive": 0.6MS, 
    "version_self_next": 0.8MS, 
    "version_peer_active": 0.7MS, 
    "version_peer_inactive": 0.6MS, 
    "version_peer_next": 0.8MS, 
    "version_nic_active": 0.7MS, 
    "version_nic_inactive": 0.6MS, 
    "version_nic_next": 0.8MS 
} 
```
RC 0 will mean success all other RCs will mean error. 

#### <a name='MuxcableFirmwareDownload'></a>4.4.2	Muxcable Firmware Download

##### 4.4.2.1	Start Firmware downloading
```
config muxcable firmware download <firmware_file> <port_name>
```

Example: 
```
sudo config muxcable firmware download AEC_WYOMING_B52Yb0_MS_0.6_20201218.bin Ethernet0
Output:
firmware download successful Ethernet0
```

##### 4.4.2.2 Check downloading status
```
show muxcable firmware version <port_name>
```

Example: 
```
admin@sonic$ show muxcable firmware download_status Ethernet52 
{ 
    "done”  
}
```

RC 0 will mean cli executed successfully.


#### <a name='ActivateFirmware'></a>4.4.3	Activate Firmware 
```
config muxcable firmware activate <port_name>
```

Example:
``` 
sudo config muxcable firmware activate Ethernet0
Output: 
firmware activate successful for Ethernet0
```

#### <a name='RollbackFirmware'></a>4.4.5 Rollback Firmware
```
config muxcable firmware rollback <port_name>
```
Example 
```
sudo config muxcable firmware rollback Ethernet0
Output:
firmware rollback successful Ethernet0
```

## 5 Modules 
### 5.1 linkmgrd
linkmgrd is a simple framework is a simple framework that enables dual ToR to switch/pull the link when it detects loss of health signs. 

linkmgrd module consists of four submodules:
1.	LinkProber submodule is responsible for monitoring the link health status. It is a simple module that will send ICMP packets and listen to ICMP reply packets. ICMP packet payload will contain information about the originating ToR. LinkProber will report the link as active if it receives ICMP reply for packets that it sends. If LinkProber receives ICMP packet sent by peer ToR, it will report the link as standby. If no response is received, LinkProber will report link status as unknown.

2.	LinkState submodule has two distinctive states: **LinkUp** and LinkDown states. It will listen to state DB for link update messages and updates its state accordingly.

3.	MuxState submodule will report current MUX state retrievable via I2C command. Expected states are: MuxActive, MuxStandby, and MuxUnknown corresponding to MUX pointing to current ToR, MUX pointing to peer ToR, and no response is received from MUX, respectively.
The above three submodule will have their states processed by LinkManager module. LinkManager will run at frequency slower that its submodule frequencies to prevent as much hysteresis as possible. 



 