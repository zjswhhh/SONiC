# SONiC Active-Standby Dual ToR HLD

## Revision

|  Rev  |   Date   |    Author     | Change Description             |
| :---: | :------: | :-----------: | ------------------------------ |
|  0.1  | 01/03/23 |  Jing Zhang   | Initial version                |

## Scope

This document provides an introduction of the high-level design of SONiC Dual ToR with Smart Y-Cable solution. 

## Content

## 1 Requirement

Switch traffic to a healthy link / ToR when there is a link failure. 

## 2 Theary of Operation 
### 2.1 Smart Y-Cable (MUX Cable) Behavior 
* From ToR to NIC both links will be up, but only active link traffic will be forwarded to the NIC side. 
* From NIC to ToR, traffic will be broadcasted to both ToR. 
* There will be no link down during mux switchovers. 
* Expect a few packets corrupted/dropped from ToR to NIC during the mux switchover.
* Expect no traffic disruption during mux switchover from NIC to ToR. 

### Routing Behavior
* Both ToR will have same VLAN configuration as well as virtual mac address. 
* Both ToR will advertise VLAN prefix to T1
* One of the ToR port will be active state, the other ToR port will be standby state. 
* For southbound traffic (from T1 to server), active port will forward it down directly to server, standby port will forward it to the peer ToR (through IPinIP tunnel). 
* For northbound traffic (from server to T1), active port forward it directly. ToR will drop all the traffic received on the standby port. 

## 3 Data Schema 
### 3.1 Config DB 
* `MUX_LINKMGR|LINK_PROBE`
  * heartbeat probe interval in millisecond, default value is 100.   
    `interval_v4: 100`
  * heartbeat probe interval for ipv6 in millisecond, default value is 1000  
    `interval_v6: 1000`
  * event count to confirm a positive state transition, i.e. __unknown__ to __active__  
    `positive_signal_count: 1`  
  * event count to confirm a negative state transtion, i.e. __active__ to __unknown__  
    `negative_signal_count: 3`  
  * heartbeats will be suspended for duration of `suspend_timer` when heartbeat state transits from __active__ to __unknown__.  
    `suspend_timer: 500`  
    \*This field is not in use anymore. suspending timeout will depend on `interval_v4` and `negative_signal_count`.  
  * interval to report heartbeat loss count to stream telemetry in heartbeat counts, minimum value is 50, default value is 300.  
    `interval_pck_loss_count_update: 300`
* `MUX_LINKMGR|MUXLOGGER`
  * log verbosity of __linkmgrd__  
    `log_verbosity: trace|debug|info|warning|error|fatal`
* `MUX_CABLE|<PORTNAME>`


