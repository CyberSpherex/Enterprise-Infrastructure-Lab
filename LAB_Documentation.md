# Access Switches - VLAN Configuration

## 1. VLAN Configuration (ACCESS1, ACCESS2, ACCESS3)

Create the core functional VLANs on the access layer switches:

```cli
Switch(config)#hostname ACCESS1
ACCESS1(config)#vlan 10
ACCESS1(config-vlan)#name IT
ACCESS1(config-vlan)#vlan 20
ACCESS1(config-vlan)#name HR
ACCESS1(config-vlan)#vlan 30
ACCESS1(config-vlan)#name SALES
ACCESS1(config)#vlan 99
ACCESS1(config-vlan)#name MANAGEMENT
```

**NOTE:** 

Repeat the identical VLAN database configuration on **`ACCESS2`** and **`ACCESS3`**.

## 2. Verification

Inspect the created VLANs to confirm database entry, operational status, and port assignments:

```cli
ACCESS1#show vlan brief 

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et0/0, Et0/1, Et0/2, Et0/3
                                                Et1/0, Et1/1, Et1/2, Et1/3
                                                Et2/0, Et2/1, Et2/2, Et2/3
                                                Et3/0, Et3/1, Et3/2, Et3/3
10   IT                               active    
20   HR                               active    
30   SALES                            active    
99   MANAGEMENT                       active    
1002 fddi-default                     act/unsup 
1003 token-ring-default               act/unsup 
1004 fddinet-default                  act/unsup 
1005 trnet-default                    act/unsup 
```

This command verifies that the custom VLANs exist in the local VLAN database, shows their status (`active`), and displays which switchports are currently assigned to each VLAN.

# Changing the Spanning three Mode to Rapid PVST+ (Rapid Per-VLAN Spanning Tree Plus) On all switches

Switching to Rapid PVST+ significantly speeds up network convergence times when a link fails or a topology changes—dropping recovery times from up to 50 seconds down to a few seconds by introducing rapid port states.
Before we do that let's check which mode the ACCESS switch is running.

```cli
ACCESS1#show spanning-tree summary 
Switch is in pvst mode
```

This switch is running Per-VLAN Spanning Tree Plus (PVST+), which means it calculates a separate spanning tree instance for every individual VLAN., let's change that to rapid spanning treee.

```cli
ACCESS1(config)#spanning-tree mode rapid-pvst
```

Let's confirm that the mode change was successfully!

```cli
ACCESS1#show spanning-tree summary 
Switch is in rapid-pvst mode
```

>***Some Tips:***
>
>==Run `spanning-tree mode rapid-pvst` on every switch in your network to ensure the entire fabric recovers instantly from failures.==
 Safe Migration Strategy:
>1. **Start at the Edge:** Change the mode on your access switches first.
>2. **Move to the Core:** Change your distribution/core switches last.
>3. **Maintenance Window:** Although STP is designed to be hitless, changing the mode causes a brief **re-convergence** which can drop a few packets.

# Configuring Acess ports & Enabling the portfast on Switches


```cli
ACCESS1(config)#interface ethernet 0/0
ACCESS1(config-if)#switchport mode access 
ACCESS1(config-if)#switchport access vlan 10
ACCESS1(config-if)#spanning-tree portfast edge

ACCESS1(config)#interface ethernet 0/1
ACCESS1(config-if)#switchport access vlan 20
ACCESS1(config-if)#spanning-tree portfast edge 

ACCESS1(config)#interface ethernet 1/2
ACCESS1(config-if)#switchport mode access 
ACCESS1(config-if)#switchport access vlan 30
ACCESS1(config-if)#spanning-tree portfast edge

ACCESS1(config)#interface ethernet 1/3
ACCESS1(config-if)#switchport mode access 
ACCESS1(config-if)#switchport access vlan 99
ACCESS1(config-if)#spanning-tree portfast edge 
ACCESS1(config-if)#end
```

- `switchport mode access`: Forces the interface to operate as a static access port, meaning it will only carry traffic for a single VLAN and will not negotiate trunking.

- `switchport access vlan 10`: Assigns the access port to VLAN 10, ensuring that any untagged traffic entering this port is placed into VLAN 10.

- `spanning-tree portfast edge`: Enables PortFast Edge on the interface, allowing it to bypass the standard listening and learning states (dropping straight to forwarding) to prevent initialization delays for end devices.

We can confirm that the ports are correctly configured and assigned there appropriate VLAN:

```cli
ACCESS1#show running-config interface ethernet 0/0
Building configuration...

Current configuration : 109 bytes
!
interface Ethernet0/0
 switchport access vlan 10
 switchport mode access
 spanning-tree portfast edge
end

```

This shows clearly that the ports are mapped to there exact VLAN as we configured:

```cli
ACCESS1#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Et1/3, Et2/0, Et2/1, Et2/2
                                                Et2/3, Et3/0, Et3/1, Et3/2
                                                Et3/3
10   IT                               active    Et0/0
20   HR                               active    Et0/1
30   SALES                            active    Et1/2
```

To verify that the edge setting is active on this interface, run this command:

```cli
ACCESS1#show spanning-tree interface ethernet 0/0

Vlan                Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
VLAN0010            Desg FWD 100       128.1    P2p Edge 

```

# Configuring The Trunk Ports & Disabling DTP on Static Trunks

```cli
ACCESS1(config)#interface range ethernet 0/2-3
ACCESS1(config-if-range)#switchport trunk encapsulation dot1q 
ACCESS1(config-if-range)#switchport mode trunk
ACCESS1(config-if-range)#switchport nonegotiate
ACCESS1(config-if-range)#exit
ACCESS1(config)#interface range ethernet 1/0-1
ACCESS1(config-if-range)#switchport trunk encapsulation dot1q
ACCESS1(config-if-range)#switchport mode trunk
ACCESS1(config-if-range)#switchport nonegotiate
ACCESS1(config-if-range)#exit
```

- `switchport trunk encapsulation dot1q`: Configures the encapsulation type for the trunk port to use IEEE 802.1Q (dot1q), which is the industry standard for tagging frames across multiple VLANs. _(Note: On many modern switches, dot1q is the default and only supported encapsulation, but explicitly defining it is a good best practice)._

- `switchport mode trunk`: Forces the interface to operate as a trunk port, allowing it to carry traffic for multiple VLANs simultaneously.

- When a port is statically configured as a trunk (`switchport mode trunk`), it continues to send Dynamic Trunking Protocol (DTP) frames by default to negotiate with the connected device.
- **Purpose:** Stops the port from sending and listening to DTP frames. **Best Practice:** Applied to **active static trunk ports** (inter-switch links, switch-to-router/firewall connections) to eliminate unnecessary protocol chatter, optimize link efficiency, and reduce overhead.

## Verifying Trunk Status and DTP Configuration

First, check that all intended ports are actively forming trunks:

```cli
ACCESS1#show interfaces trunk

Port        Mode             Encapsulation  Status        Native vlan
Et0/2       on               802.1q         trunking      1
Et0/3       on               802.1q         trunking      1
Et1/0       on               802.1q         trunking      1
Et1/1       on               802.1q         trunking      1
```

Next, inspect a specific interface to confirm that DTP) has been disabled:

```cli
ACCESS1#show interfaces ethernet 0/2 switchport 
Name: Et0/2
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 1 (default)
Administrative Native VLAN tagging: enabled
```

These commands show that the ports are successfully operating as trunks and that DTP negotiation is disabled (`Negotiation of Trunking: Off`).
# Disabling unused ports:

For security reasons, it's recommended to shutdown the ports that are not in use on the production network.

```cli
ACCESS1(config)#interface range ethernet 2/0-3
ACCESS1(config-if-range)#shutdown 
ACCESS1(config-if-range)#interface range ethernet 3/0-3
ACCESS1(config-if-range)#shutdown 
```

We can verify those port are shutdown with this command:

```cli
ACCESS1(config-if-range)#do show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
Ethernet0/0            unassigned      YES unset  up                    up      
Ethernet0/1            unassigned      YES unset  up                    up      
Ethernet0/2            unassigned      YES unset  up                    up      
Ethernet0/3            unassigned      YES unset  up                    up      
Ethernet1/0            unassigned      YES unset  up                    up      
Ethernet1/1            unassigned      YES unset  up                    up      
Ethernet1/2            unassigned      YES unset  up                    up      
Ethernet1/3            unassigned      YES unset  up                    up      
Ethernet2/0            unassigned      YES unset  administratively down down    
Ethernet2/1            unassigned      YES unset  administratively down down    
Ethernet2/2            unassigned      YES unset  administratively down down    
Ethernet2/3            unassigned      YES unset  administratively down down    
Ethernet3/0            unassigned      YES unset  administratively down down    
Ethernet3/1            unassigned      YES unset  administratively down down    
Ethernet3/2            unassigned      YES unset  administratively down down    
Ethernet3/3            unassigned      YES unset  administratively down down    
Vlan1                  unassigned      YES unset  administratively down down    
```

# Configuring LACP EtherChannel

## 1. Configure the Logical Port-Channel 

Always configure layer parameters (trunking, encapsulation, nonegotiate) on the logical port-channel interface *before* assigning physical ports to the channel group. This prevents compatibility errors and automatic suspension of physical links.

### Between ACCESS2 and ACCESS1

```cli
ACCESS2(config)#interface port-channel 1
ACCESS2(config-if)#switchport trunk encapsulation dot1q
ACCESS2(config-if)#switchport mode trunk
ACCESS2(config-if)#switchport nonegotiate
```
### Between ACCESS2 and ACCESS3

```cli
ACCESS2(config)#interface port-channel 2
ACCESS2(config-if)#switchport trunk encapsulation dot1q
ACCESS2(config-if)#switchport mode trunk
ACCESS2(config-if)#switchport nonegotiate
```

## 2. Assign Physical Interfaces to the Group

### Between ACCESS2 and ACCESS1

```cli
ACCESS2(config)#interface range ethernet 1/0-1
ACCESS2(config-if-range)#channel-group 1 mode active
ACCESS2(config-if-range)#channel-protocol lacp
```

### Between ACCESS2 and ACCESS3

```cli
ACCESS2(config)#interface range ethernet 1/2-3
ACCESS2(config-if-range)#channel-group 2 mode active
ACCESS2(config-if-range)#channel-protocol lacp
```
## Verification Command

```cli
ACCESS2#show etherchannel summary
```

*(Verify that the Port-Channel status shows 'SU' [Layer 2, Up] and that physical members show 'P' [Bundled in Port-channel]).*

***Other Usefull Commands to Run:***

```cli
ACCESS2#show interfaces port-channel 1 etherchannel
```

```cli
ACCESS2#show spanning-tree
```

```cli
ACCESS2#show interfaces trunk 
```

## Key Takeaway: Port-Channel Configuration

> **The Golden Rule of Port-Channels:** Configuration flows **down** from the logical parent interface (Port-Channel) to the physical child interfaces (member ports), not the other way around.


> **Parent-First Rule:** Always apply trunking, encapsulation, and security settings (`switchport mode trunk`, `switchport nonegotiate`, etc.) to the logical `port-channel` interface **before** adding physical ports to the `channel-group`.


> **Why it matters:** If you configure the physical ports first, they will conflict with the unconfigured (default) port-channel parent, causing a parameter mismatch and triggering the `%EC-5-CANNOT_BUNDLE2` suspension error. Configuring the parent first ensures the switch automatically pushes down matching parameters to all member links cleanly.

***Note:*** 

>*The exact same configuration were replicated across ACCESS1 and ACCESS3.*

# Configuring The CORE1 and CORE2 Switches

In this demo, we're going to configure the CORE1 and CORE2 switches just like we did with the ACCESS switches. We're gonna configure these protocols too: VLANs, trunking interfaces, EtherChannel, locking down unused ports, and disabling DTP.

Let's start with the CORE1 switch, and note that the same configuration should be applied on the CORE2 switch as well.

## 1. Basic Setup & Hostname

```cli
CORE1#configure terminal
CORE1(config)#hostname CORE1
```

## 2. Configure RSTP

```cli
CORE1(config)#do show span sum   
Switch is in rapid-pvst mode
```

The CORE1 switch already configured to use Rapid spanning three + (RSTP), no need to configure it.

## 3. VLAN Creation

```cli
CORE1(config)#vlan 10
CORE1(config-vlan)#name IT
CORE1(config-vlan)#vlan 20
CORE1(config-vlan)#name HR
CORE1(config-vlan)#vlan 30
CORE1(config-vlan)#name SALES
CORE1(config-vlan)#vlan 99
CORE1(config-vlan)#name MANAGEMENT 
CORE1(config-vlan)#exit
```

## Configuring the Trunk Interfaces

__(Note: Here we configured the duplex to full because there are duplex mismatches on user-facing or inter-layer links between the access switches and core switches.)__

```cli
CORE1(config)#interface range ethernet 0/0-3
CORE1(config-if-range)#switchport trunk encapsulation dot1q 
CORE1(config-if-range)#switchport mode trunk 
CORE1(config-if-range)#switchport nonegotiate 
CORE1(config-if-range)#duplex full
CORE1(config-if-range)#shutdown
CORE1(config-if-range)#no shutdown
```

Use the commands `shutdown` / `no shutdown` to bounce the interface if a trunk doesn't form immediately after changing interface settings (like duplex). If `show interfaces trunk` does not show the trunk, bouncing the port forces the link to reset and re-negotiate with the new settings.

```cli
CORE1(config)#interface range ethernet 1/1-3
CORE1(config-if-range)#switchport trunk encapsulation dot1q 
CORE1(config-if-range)#switchport mode trunk 
CORE1(config-if-range)#switchport nonegotiate 
CORE1(config-if-range)#duplex full
CORE1(config-if-range)#shutdown
CORE1(config-if-range)#no shutdown
```

## 4. Port-Channel (LACP) Configuration

Configuring the logical Port-Channel interface.

```cli
CORE1(config)#interface port-channel 1
CORE1(config-if)#switchport trunk encapsulation dot1q
CORE1(config-if)#switchport mode trunk
CORE1(config-if)#switchport nonegotiate
CORE1(config-if)#exit
```

Applying the Port-Channel created above to the 4 member interfaces connecting CORE1 and CORE2.

```cli
CORE1(config)#interface range ethernet 0/0-3
CORE1(config-if-range)#channel-protocol lacp
CORE1(config-if-range)#channel-group 1 mode active
CORE1(config-if-range)#end
```

Verifying that the EtherChannel is up and running using the `show lacp neighbor` command.

```cli
CORE1#show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs 
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode       P - Device is in Passive mode     

Channel group 1 neighbors

Partner's information:

                  LACP port                        Admin  Oper   Port    Port
Port      Flags   Priority  Dev ID          Age    key    Key    Number  State
Et0/0     SA      32768     aabb.cc80.0200   8s    0x0    0x1    0x1     0x3D  
Et0/1     SA      32768     aabb.cc80.0200  16s    0x0    0x1    0x2     0x3D  
Et0/2     SA      32768     aabb.cc80.0200   0s    0x0    0x1    0x3     0x3D  
Et0/3     SA      32768     aabb.cc80.0200  21s    0x0    0x1    0x4     0x3D  
CORE1#
```

## Locking Down Unused Ports

```cli
CORE1(config)#interface range ethernet 2/0-3
CORE1(config-if-range)#shutdown 
CORE1(config)#interface range ethernet 3/0-3
CORE1(config-if-range)#shutdown 
```

# Configuring STP on CORE1 and CORE2

Instead of keeping `CORE2` completely idle for all traffic, splitting the primary root bridge duties allows you to utilize the bandwidth on both core switches simultaneously:

- **CORE1** acts as the primary root for VLANs 10 and 20, while serving as the backup for VLAN 30.
    
- **CORE2** acts as the primary root for VLAN 30, while serving as the backup for VLANs 10 and 20.

## On CORE1:

```cli
CORE1# configure terminal
CORE1(config)# spanning-tree vlan 10,20,99 priority 4096
CORE1(config)# spanning-tree vlan 30 priority 8192
CORE1(config)# end
```

## On CORE2:

```cli
CORE2# configure terminal
CORE2(config)# spanning-tree vlan 30 priority 4096
CORE2(config)# spanning-tree vlan 10,20,99 priority 8192
CORE2(config)# end
```

# Configuring SVI, Inter-VLAN and HSRP on CORE1 & and CORE2 Switches

### 1. VLAN Subnet & HSRP IP Allocation Plan

|**VLAN**|**Name**|**CORE1 SVI IP**|**CORE2 SVI IP**|**HSRP Virtual IP (VIP)**|**HSRP Active**|
|---|---|---|---|---|---|
|**VLAN 10**|IT|`192.168.10.2/24`|`192.168.10.3/24`|**`192.168.10.1/24`**|**CORE1**|
|**VLAN 20**|HR|`192.168.20.2/24`|`192.168.20.3/24`|**`192.168.20.1/24`**|**CORE1**|
|**VLAN 30**|Sales|`192.168.30.2/24`|`192.168.30.3/24`|**`192.168.30.1/24`**|**CORE2**|
|**VLAN 99**|Management|`192.168.99.2/24`|`192.168.99.3/24`|**`192.168.99.1/24`**|**CORE1**|

#### Planning Notes

- **Gateway Consistency**: The Virtual IP (VIP) ending in `.1` serves as the default gateway address configured on all end-user hosts within each respective subnet.
    
- **IP Separation**:
    
    - CORE1 uses `.2` for all physical SVI interfaces.
        
    - CORE2 uses `.3` for all physical SVI interfaces.
        
- **Active-Active Load Sharing**:
    
    - CORE1 acts as primary active default gateway for **VLAN 10**, **VLAN 20**, and **VLAN 99**.
        
    - CORE2 acts as primary active default gateway for **VLAN 30**, distributing traffic handling across both core switches.
        
## 2. CORE1 SVI & HSRP Configuration

This configuration enables Layer 3 inter-VLAN routing on **CORE1** and configures High Availability for default gateways using **HSRP Version 2**.

- **Active Role (Priority 110):** CORE1 serves as the active default gateway for VLAN 10 (IT), VLAN 20 (HR), and VLAN 99 (Management).
    
- **Standby Role (Priority 90):** CORE1 acts as the backup default gateway for VLAN 30 (Sales).
    
- **Preemption:** Enabled across all groups so CORE1 automatically re-claims the active role upon reboot/recovery.

```cli
CORE1# configure terminal
CORE1(config)# ip routing

! --- VLAN 10 (IT) - Active on CORE1 ---
CORE1(config)# interface Vlan10
CORE1(config-if)# ip address 192.168.10.2 255.255.255.0
CORE1(config-if)# standby version 2
CORE1(config-if)# standby 10 ip 192.168.10.1
CORE1(config-if)# standby 10 priority 110
CORE1(config-if)# standby 10 preempt
CORE1(config-if)# no shutdown

! --- VLAN 20 (HR) - Active on CORE1 ---
CORE1(config)# interface Vlan20
CORE1(config-if)# ip address 192.168.20.2 255.255.255.0
CORE1(config-if)# standby version 2
CORE1(config-if)# standby 20 ip 192.168.20.1
CORE1(config-if)# standby 20 priority 110
CORE1(config-if)# standby 20 preempt
CORE1(config-if)# no shutdown

! --- VLAN 30 (Sales) - Standby on CORE1 (Active on CORE2) ---
CORE1(config)# interface Vlan30
CORE1(config-if)# ip address 192.168.30.2 255.255.255.0
CORE1(config-if)# standby version 2
CORE1(config-if)# standby 30 ip 192.168.30.1
CORE1(config-if)# standby 30 priority 90
CORE1(config-if)# standby 30 preempt
CORE1(config-if)# no shutdown

! --- VLAN 99 (Management) ---
CORE1(config)# interface Vlan99
CORE1(config-if)# ip address 192.168.99.2 255.255.255.0
CORE1(config-if)# standby version 2
CORE1(config-if)# standby 99 ip 192.168.99.1
CORE1(config-if)# standby 99 priority 110
CORE1(config-if)# standby 99 preempt
CORE1(config-if)# no shutdown
CORE1(config-if)# end
```

#### Configuration Notes:

- **`ip routing`**: Enables Layer 3 inter-VLAN routing globally on CORE1.
    
- **IP Addressing Plan**:
    
    - Physical SVI IPs use `.2` (`192.168.x.2`).
        
    - Virtual IPs (VIPs) use `.1` (`192.168.x.1`), serving as the default gateways for end hosts.
        
- **HSRP Version**: Uses `standby version 2` across all interfaces to support higher group numbers and dedicated multicast addressing (`224.0.0.102`).


## 3. CORE2 SVI & HSRP Configuration

```cli
CORE2# configure terminal
CORE2(config)# ip routing

! --- VLAN 10 (IT) - Standby on CORE2 ---
CORE2(config)# interface Vlan10
CORE2(config-if)# ip address 192.168.10.3 255.255.255.0
CORE2(config-if)# standby version 2
CORE2(config-if)# standby 10 ip 192.168.10.1
CORE2(config-if)# standby 10 priority 90
CORE2(config-if)# standby 10 preempt
CORE2(config-if)# no shutdown

! --- VLAN 20 (HR) - Standby on CORE2 ---
CORE2(config)# interface Vlan20
CORE2(config-if)# ip address 192.168.20.3 255.255.255.0
CORE2(config-if)# standby version 2
CORE2(config-if)# standby 20 ip 192.168.20.1
CORE2(config-if)# standby 20 priority 90
CORE2(config-if)# standby 20 preempt
CORE2(config-if)# no shutdown

! --- VLAN 30 (Sales) - Active on CORE2 ---
CORE2(config)# interface Vlan30
CORE2(config-if)# ip address 192.168.30.3 255.255.255.0
CORE2(config-if)# standby version 2
CORE2(config-if)# standby 30 ip 192.168.30.1
CORE2(config-if)# standby 30 priority 110
CORE2(config-if)# standby 30 preempt
CORE2(config-if)# no shutdown

! --- VLAN 99 (Management) ---
CORE2(config)# interface Vlan99
CORE2(config-if)# ip address 192.168.99.3 255.255.255.0
CORE2(config-if)# standby version 2
CORE2(config-if)# standby 99 ip 192.168.99.1
CORE2(config-if)# standby 99 priority 90
CORE2(config-if)# standby 99 preempt
CORE2(config-if)# no shutdown
CORE2(config-if)# end
```

#### Configuration Notes:

- **`ip routing`**: Enables Layer 3 inter-VLAN routing globally on CORE2.
    
- **IP Addressing Plan**:
    
    - Physical SVI IPs use `.3` (`192.168.x.3`).
        
    - Virtual IPs (VIPs) use `.1` (`192.168.x.1`), matching CORE1 to provide a unified default gateway to clients.
        
- **HSRP Version**: Uses `standby version 2` across all interfaces to align with CORE1 and avoid version mismatch issues.
    
- **Active vs. Standby Roles**:
    
    - **Priority 110 (Active):** VLAN 30 (Sales) is primary on CORE2.
        
    - **Priority 90 (Standby):** VLANs 10, 20, and 99 act as backups (CORE1 is Active with priority 110).
        
- **Preemption (`standby X preempt`)**: Enables CORE2 to automatically assume the Active role for VLAN 30 whenever it is operational, and dynamically take over VLANs 10, 20, and 99 if CORE1 fails.

# Configuring DHCP on CORE1 and CORE2

## Step 1: Configure CORE1 (Primary Pool - First Half of Subnet)

```cli
CORE1# configure terminal

! --- Exclude Virtual IP, Core Switch IPs, and CORE2's DHCP Pool Range ---
CORE1(config)#ip dhcp excluded-address 192.168.10.1 192.168.10.3
CORE1(config)#ip dhcp excluded-address 192.168.10.128 192.168.10.254

CORE1(config)#ip dhcp excluded-address 192.168.20.1 192.168.20.3
CORE1(config)#ip dhcp excluded-address 192.168.20.128 192.168.20.254

CORE1(config)#ip dhcp excluded-address 192.168.30.1 192.168.30.3
CORE1(config)#ip dhcp excluded-address 192.168.30.128 192.168.30.254

CORE1(config)#ip dhcp excluded-address 192.168.99.1 192.168.99.3
CORE1(config)#ip dhcp excluded-address 192.168.99.128 192.168.99.254

! --- Pool Definitions (Serves .4 through .127) ---
CORE1(config)#ip dhcp pool VLAN10_IT
CORE1(dhcp-config)#network 192.168.10.0 255.255.255.0
CORE1(dhcp-config)#default-router 192.168.10.1
CORE1(dhcp-config)#dns-server 8.8.8.8

CORE1(config)#ip dhcp pool VLAN20_HR
CORE1(dhcp-config)#network 192.168.20.0 255.255.255.0
CORE1(dhcp-config)#default-router 192.168.20.1
CORE1(dhcp-config)#dns-server 8.8.8.8

CORE1(config)#ip dhcp pool VLAN30_SALES
CORE1(dhcp-config)#network 192.168.30.0 255.255.255.0
CORE1(dhcp-config)#default-router 192.168.30.1
CORE1(dhcp-config)#dns-server 8.8.8.8

CORE1(config)# ip dhcp pool MANAGEMENT_POOL
CORE1(dhcp-config)# network 192.168.99.0 255.255.255.0
CORE1(dhcp-config)# default-router 192.168.99.1
CORE1(dhcp-config)# dns-server 8.8.8.8
CORE1(dhcp-config)# end
CORE1# write memory
```

#### Configuration Notes:

- **Split-Scope Allocation**: CORE1 is assigned the lower half of the usable host range (`.4` through `.127`), providing 124 available leases per VLAN.
    
- **IP Exclusions (`ip dhcp excluded-address`)**:
    
    - Protects static infrastructure addresses (`.1` VIP, `.2` CORE1 SVI, `.3` CORE2 SVI).
        
    - Excludes `.128` through `.254` to prevent lease overlap with CORE2's DHCP pool.
        
- **Default Router Assignment**: Sets `default-router` to the HSRP Virtual IP (`.1`) across all pools, ensuring hosts maintain gateway redundancy even if a core switch goes down.
    
- **DNS Configuration**: Pushes `8.8.8.8` to clients as the primary domain name server.

## Step 2: Configure CORE2 (Secondary Pool - Second Half of Subnet)

```cli
CORE2# configure terminal

! --- Exclude Virtual IP, Core Switch IPs, and CORE1's DHCP Pool Range ---
CORE2(config)#ip dhcp excluded-address 192.168.10.1 192.168.10.127
CORE2(config)#ip dhcp excluded-address 192.168.20.1 192.168.20.127
CORE2(config)#ip dhcp excluded-address 192.168.30.1 192.168.30.127
CORE2(config)#ip dhcp excluded-address 192.168.99.1 192.168.99.127

! --- Pool Definitions (Serves .128 through .254) ---
CORE2(config)#ip dhcp pool VLAN10_IT
CORE2(dhcp-config)#network 192.168.10.0 255.255.255.0
CORE2(dhcp-config)#default-router 192.168.10.1
CORE2(dhcp-config)# dns-server 8.8.8.8

CORE2(config)#ip dhcp pool VLAN20_HR
CORE2(dhcp-config)#network 192.168.20.0 255.255.255.0
CORE2(dhcp-config)#default-router 192.168.20.1
CORE2(dhcp-config)#dns-server 8.8.8.8

CORE2(config)#ip dhcp pool VLAN30_SALES
CORE2(dhcp-config)#network 192.168.30.0 255.255.255.0
CORE2(dhcp-config)#default-router 192.168.30.1
CORE2(dhcp-config)#dns-server 8.8.8.8

CORE2(config)# ip dhcp pool MANAGEMENT_POOL
CORE2(dhcp-config)# network 192.168.99.0 255.255.255.0
CORE2(dhcp-config)# default-router 192.168.99.1
CORE2(dhcp-config)# dns-server 8.8.8.8
CORE2(dhcp-config)# end
CORE2# write memory
```

### Configuration Notes:

- **Split-Scope Allocation**: CORE2 is assigned the upper half of the usable host range (`.128` through `.254`), providing 127 available leases per VLAN without requiring dynamic DHCP server coordination.
    
- **IP Exclusions (`ip dhcp excluded-address`)**:
    
    - Excludes `.1` through `.127` to protect the static infrastructure addresses (`.1` VIP, `.2` CORE1 SVI, `.3` CORE2 SVI) and prevent address conflicts with CORE1's DHCP pool.
        
- **Default Router Assignment**: Configures `default-router` as the shared HSRP Virtual IP (`.1`) across all pools so hosts automatically maintain gateway redundancy regardless of which core switch issued the lease.
    
- **DNS Configuration**: Pushes `8.8.8.8` to clients as the primary domain name server.

# HSRP & Spanning Tree failover and redundancy test

## CLI Verification Commands

I'm gonna run these commands on both core switches to verify initial role allocation before running failover tests:

```cli
! --- Verify HSRP Group Roles & Virtual IP Status ---
CORE1# show standby brief
CORE2# show standby brief

! --- Inspect Specific VLAN HSRP State & Virtual MAC ---
CORE1# show standby vlan 10

! --- Verify STP Root Bridge Status & Active Mode ---
CORE1# show spanning-tree summary
CORE1# show spanning-tree vlan 10
```

These are the results on CORE1 and CORE2 switches:

### CORE1:

```cli
CORE1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   110 P Active  local           192.168.10.3    192.168.10.1
Vl20        20   110 P Active  local           192.168.20.3    192.168.20.1
Vl30        30   90  P Standby 192.168.30.3    local           192.168.30.1
Vl99        99   110 P Active  local           192.168.99.3    192.168.99.1

CORE1#show spanning-tree vla 10,20

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     aabb.cc00.0600
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106   (priority 4096 sys-id-ext 10)
             Address     aabb.cc00.0600
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 
Po1                 Desg FWD 41        128.65   Shr 


VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    4116
             Address     aabb.cc00.0600
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4116   (priority 4096 sys-id-ext 20)
             Address     aabb.cc00.0600
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 
Po1                 Desg FWD 41        128.65   Shr 
```

### CORE2:

```cli
CORE2#show standby brief 
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   90  P Standby 192.168.10.2    local           192.168.10.1
Vl20        20   90  P Standby 192.168.20.2    local           192.168.20.1
Vl30        30   110 P Active  local           192.168.30.2    192.168.30.1
Vl99        99   90  P Standby 192.168.99.2    local           192.168.99.1

CORE2#show spanning-tree vla 10,20

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     aabb.cc00.0600
             Cost        41
             Port        65 (Port-channel1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8202   (priority 8192 sys-id-ext 10)
             Address     aabb.cc00.0700
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 
Po1                 Root FWD 41        128.65   Shr 

VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    4116
             Address     aabb.cc00.0600
             Cost        41
             Port        65 (Port-channel1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8212   (priority 8192 sys-id-ext 20)
             Address     aabb.cc00.0700
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 
Po1                 Root FWD 41        128.65   Shr 
```

## Failover & Convergence Test Results

### Test Execution Procedure

1. Start a Continuous Ping on an End Host `IT-PC1` (`192.168.10.4`) to host on VLAN 20 (`192.168.20.4`).

```bash
IT-PC1> ping 192.168.20.4 -t
```

2. Simulated core failure by shutting down the **`CORE1`** switch.
3. Monitored syslog state changes on `CORE2` and ping continuity on `IT-PC1`.

### Observed Results

**HSRP Gateway Failover:** `CORE2` detected missing hellos from `CORE1` and successfully promoted itself to **Active** for VLANs 10, 20, and 99.

**Console Logs (`CORE2`):**

```cli
%HSRP-5-STATECHANGE: Vlan10 Grp 10 state Standby -> Active
%HSRP-5-STATECHANGE: Vlan20 Grp 20 state Standby -> Active
%HSRP-5-STATECHANGE: Vlan99 Grp 99 state Standby -> Active
```

**STP Convergence Performance and Failover :**
  
- With **Rapid PVST+ (`802.1w`)** enabled, Layer 2 path failover occurred via active proposal/agreement handshakes rather than legacy timers. When `CORE1` failed, `CORE2` assumed the Root Bridge role for VLANs 10, 20, and 99, seamlessly unblocking the backup trunk path on `ACCESS1`.
- Measured packet loss during full failover: **1–2 pings dropped (~1–3 seconds total outage)**.

**Preemption Verification:** Restoring the `CORE1` Switch caused `CORE1` to instantly reclaim the **Active** HSRP gateway role and **STP Root Bridge** status without manual intervention.

```cli
CORE2#show spanning-tree vlan 10,20 

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    8202
             Address     aabb.cc00.0700
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8202   (priority 8192 sys-id-ext 10)
             Address     aabb.cc00.0700
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 

VLAN0020
  Spanning tree enabled protocol rstp
  Root ID    Priority    8212
             Address     aabb.cc00.0700
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8212   (priority 8192 sys-id-ext 20)
             Address     aabb.cc00.0700
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr

VLAN0099
  Spanning tree enabled protocol rstp
  Root ID    Priority    8291
             Address     aabb.cc00.0700
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    8291   (priority 8192 sys-id-ext 99)
             Address     aabb.cc00.0700
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Et1/1               Desg FWD 100       128.6    Shr 
Et1/2               Desg FWD 100       128.7    Shr 
Et1/3               Desg FWD 100       128.8    Shr 
```
## Key Takeaways & Design Validation

- **Layer 3 Redundancy:** HSRPv2 ensures end hosts always reach their default gateway (`.1`), regardless of which physical core switch is online.
    
- **Layer 2 Optimization:** Rapid PVST+ eliminates legacy 802.1D _Listening/Learning_ delays (30–60s), ensuring link failover matches HSRP speed.
    
- **Active-Active Load Sharing:** Splitting active HSRP priorities between `CORE1` (VLANs 10, 20, 99) and `CORE2` (VLAN 30) optimizes core bandwidth utilization under normal operating conditions.

# HQ Core to Edge Routing Configuration

## Overview

Establishes Layer 3 routed point-to-point transit links between the Core switches (`CORE1`/`CORE2`) and Edge routers (`EDGE1`/`EDGE2`).
## Interface & IP Allocation Table

| **Link Segment**                      | **Subnet**    | **Local Device & Interface**    | **Remote Device & Interface**       |
| ------------------------------------- | ------------- | ------------------------------- | ----------------------------------- |
| **CORE1 Link**                        | `10.0.1.0/30` | `CORE1` (e1/0) — `10.0.1.1/30`  | `EDGE1` (Gi0/0) — `10.0.1.2/30`     |
| **CORE2 Link**                        | `10.0.2.0/30` | `CORE2` (e1/0) — `10.0.2.1/30`  | `EDGE2` (Gi0/0) — `10.0.2.2/30`     |
| **EDGE Cross-Link**                   | `10.0.3.0/30` | `EDGE1` (Gi0/1) — `10.0.3.1/30` | `EDGE2` (Gi0/1) — `10.0.3.2/30`     |
| **EDGE1 $\leftrightarrow$ ISP-EDGE1** | `10.1.1.0/30` | `EDGE1` (Gi0/2) — `10.1.1.2/30` | `ISP-EDGE1` (Gi0/0) — `10.1.1.1/30` |
| **EDGE2 $\leftrightarrow$ ISP-EDGE1** | `10.1.2.0/30` | `EDGE2` (Gi0/2) — `10.1.2.2/30` | `ISP-EDGE1` (Gi0/1) — `10.1.2.1/30` |

---

## Device Configuration Steps

### 1. CORE1 (e1/0 -> EDGE1)

```cli
CORE1# configure terminal
CORE1(config)#interface ethernet 1/0
CORE1(config-if)# description Transit Link to EDGE1 (Gi0/0)
CORE1(config-if)#no switchport 
CORE1(config-if)#ip address 10.0.1.1 255.255.255.252
CORE1(config-if)#no shutdown 
CORE1(config-if)#end
```

### 2. EDGE1 (Gi0/0 -> CORE1)

```cli
EDGE1# configure terminal
EDGE1(config)#interface gigabitEthernet 0/0
EDGE1(config-if)#description Transit Link to CORE1 (e1/0)
EDGE1(config-if)#ip address 10.0.1.2 255.255.255.252
EDGE1(config-if)#no shutdown 
EDGE1(config-if)#end
```

### 3. CORE2 (e1/0 -> EDGE2)

```cli
CORE2# configure terminal
CORE2(config)#interface ethernet 1/0
CORE2(config-if)#description Transit Link to EDGE2 (Gi0/0)
CORE2(config-if)#no switchport 
CORE2(config-if)#ip address 10.0.2.1 255.255.255.0
CORE2(config-if)#no shutdown 
CORE2(config-if)#end
```

### 4. EDGE2 (Gi0/0 -> CORE2)

```cli
EDGE2# configure terminal
EDGE2(config)#interface gigabitEthernet 0/0
EDGE2(config-if)#description Transit Link to CORE2 (e1/0)
EDGE2(config-if)#ip address 10.0.2.2 255.255.255.252
EDGE2(config-if)#no shutdown 
EDGE2(config-if)#end
```

### EDGE1 (Gi0/1 -> EDGE2)

```cli
EDGE1# configure terminal
EDGE1(config)#interface gigabitEthernet 0/1
EDGE1(config-if)#description Transit Cross-Link to EDGE2 (Gi0/1)
EDGE1(config-if)#ip address 10.0.3.1 255.255.255.252
EDGE1(config-if)#no shutdown 
EDGE1(config-if)#end
```

### EDGE2 (Gi0/1 -> EDGE1)

```cli
EDGE2# configure terminal
EDGE2(config)#interface gigabitEthernet 0/1
EDGE2(config-if)#description Transit Cross-Link to EDGE1 (Gi0/1)
EDGE2(config-if)#ip address 10.0.3.2 255.255.255.252
EDGE2(config-if)#no shutdown 
EDGE2(config-if)#end
```

### ISP-EDGE1 (GigabitEthernet 0/0 & GigabitEthernet 0/1)

```cli
ISP-EDGE1# configure terminal

! --- Link to EDGE1 ---

ISP-EDGE1(config)#interface gigabitEthernet 0/0
ISP-EDGE1(config-if)#description Link to EDGE1 (Gi0/2)
ISP-EDGE1(config-if)#ip address 10.1.1.1 255.255.255.252
ISP-EDGE1(config-if)#no shutdown 
ISP-EDGE1(config-if)#exit 

! --- Link to EDGE2 ---

ISP-EDGE1(config)#interface gigabitEthernet 2 
ISP-EDGE1(config-if)#description Link to EDGE2 (Gi0/2)
ISP-EDGE1(config-if)#ip address 10.1.2.1 255.255.255.252
ISP-EDGE1(config-if)#no shutdown 
ISP-EDGE1(config-if)#end
```

### EDGE1 & EDGE2 (Uplink Interfaces)

```cli
EDGE1(config)# interface GigabitEthernet0/2
EDGE1(config-if)# description Uplink to ISP-EDGE1 (Gi0/0)
EDGE1(config-if)# ip address 10.1.1.2 255.255.255.252
EDGE1(config-if)# no shutdown

EDGE2(config)# interface GigabitEthernet0/2
EDGE2(config-if)# description Uplink to ISP-EDGE1 (Gi0/1)
EDGE2(config-if)# ip address 10.1.2.2 255.255.255.252
EDGE2(config-if)# no shutdown
```

### Verification Commands

Run a test ping to confirm connectivity:

```cli
! --- Verify Physical State & IP Assignments ---
show ip interface brief | exclude unassigned

! --- Test Point-to-Point Reachability ---
CORE1# ping 10.0.1.2
CORE2# ping 10.0.2.1
```

Here is a complete, production-grade documentation section formatted in Clean Markdown—ready to paste directly into Obsidian and publish on GitHub.


# Campus Core & Edge Dynamic Routing (EIGRP AS 100)

## Overview & Architectural Intent

To achieve high availability, low-latency convergence, and seamless traffic distribution across the campus core and WAN edge, **EIGRP Autonomous System 100** was deployed across the routed L3 transit infrastructure (`CORE1`, `CORE2`, `EDGE1`, `EDGE2`, and `ISP-EDGE1`).

### Key Design Decisions

- **Pure Layer 3 Transit:** All interconnecting links utilize tight `/30` point-to-point subnets (`255.255.255.252`), eliminating Spanning Tree Protocol (STP) dependency, broadcast domains, and First Hop Redundancy Protocol (FHRP) overhead between core and edge devices.
  
- **Equal-Cost Multi-Pathing (ECMP):** By advertising dual symmetric L3 paths through `EDGE1` and `EDGE2`, internal devices natively perform flow-based load balancing and sub-second convergence upon link failure.

- **Classical EIGRP Implementation:** Explicit wildcard masks (`0.0.0.3`) and disabled automatic summarization (`no auto-summary`) ensure exact matching on point-to-point interfaces, preventing unexpected route aggregation across subnets.

## Device Configuration Steps

### 1. CORE1 (Campus Core Alpha)


```cli
! Enable EIGRP process 100 for internal Core-to-Edge link

CORE1(config)#router eigrp 100
CORE1(config-router)#no auto-summary

! Transit Link to EDGE1
CORE1(config-router)#network 10.0.1.0 0.0.0.3

! Campus VLAN Subnets
CORE1(config-router)#network 192.168.10.0 0.0.0.255
CORE1(config-router)#network 192.168.20.0 0.0.0.255
CORE1(config-router)#network 192.168.30.0 0.0.0.255
CORE1(config-router)#network 192.168.99.0 0.0.0.255

! Passive Interfaces for Client SVIs
CORE1(config-router)#passive-interface vlan 10
CORE1(config-router)#passive-interface vlan 20
CORE1(config-router)#passive-interface vlan 30
CORE1(config-router)#passive-interface vlan 99
CORE1(config-router)#end
```

### 2. CORE2 (Campus Core Beta)


```cli
! Enable EIGRP process 100 for internal Core-to-Edge link

CORE2(config)#router eigrp 100
CORE2(config-router)#no auto-summary

! Transit Link to EDGE2
CORE2(config-router)#network 10.0.2.0 0.0.0.3

! Campus VLAN Subnets
CORE2(config-router)#network 192.168.10.0 0.0.0.255
CORE2(config-router)#network 192.168.20.0 0.0.0.255
CORE2(config-router)#network 192.168.30.0 0.0.0.255
CORE2(config-router)#network 192.168.99.0 0.0.0.255

! Passive Interfaces for Client SVIs
CORE2(config-router)#passive-interface vlan 10
CORE2(config-router)#passive-interface vlan 20
CORE2(config-router)#passive-interface vlan 30
CORE2(config-router)#passive-interface vlan 99
CORE2(config-router)#end
```

### 3. EDGE1 (WAN Edge Router 1)


```cli
! Enable EIGRP process 100 across Core, Cross-Link, and WAN Boundary

EDGE1(config)#router eigrp 100
EDGE1(config-router)#no auto-summary 
EDGE1(config-router)#network 10.0.1.0 0.0.0.3
EDGE1(config-router)#network 10.0.3.0 0.0.0.3
EDGE1(config-router)#network 10.1.1.0 0.0.0.3
```

### 4. EDGE2 (WAN Edge Router 2)


```cli
! Enable EIGRP process 100 across Core, Cross-Link, and WAN Boundary

EDGE2(config)#router eigrp 100
EDGE2(config-router)#no auto-summary 
EDGE2(config-router)#network 10.0.2.0 0.0.0.3
EDGE2(config-router)#network 10.0.3.0 0.0.0.3
EDGE2(config-router)#network 10.1.2.0 0.0.0.3
```

### 5. ISP-EDGE1 (WAN Boundary Aggregation)


```cli
! Enable EIGRP process 100 facing internal Edge Routers

ISP-EDGE1(config)#router eigrp 100
ISP-EDGE1(config-router)#no auto-summary 
ISP-EDGE1(config-router)#network 10.1.1.0 0.0.0.3
ISP-EDGE1(config-router)#network 10.1.2.0 0.0.0.3
ISP-EDGE1(config-router)#end
```


### EIGRP Command Syntax Explanation

##### `router eigrp 100`

- Initializes the Enhanced Interior Gateway Routing Protocol (EIGRP) routing process on the device.
    
- The number `100` represents the **Autonomous System (AS) Number**. For two or more Cisco routers to establish an EIGRP neighbor relationship and exchange routing updates, their AS numbers **must match exactly**.

##### `no auto-summary`

- Disables automatic classful route aggregation at network boundaries.
    
- By default (in older IOS versions), EIGRP automatically summarizes subnets to their classful boundaries (Class A: `/8`, Class B: `/16`, Class C: `/24`) when crossing network boundaries. Disabling this ensures that specific subnet masks (such as our `/30` point-to-point transit links) are advertised verbatim, preventing blackhole routing and asymmetric traffic paths.

##### `network 10.0.1.0 0.0.0.3`

Enables EIGRP on specific physical/logical interfaces and advertises those attached subnets into the routing table.

- The **IP address** (`10.0.1.0`) specifies the target network block.
        
- The **Wildcard Mask** (`0.0.0.3`) acts as an inverse subnet mask (`255.255.255.252`). It tells the router to evaluate the exact `/30` interface range (`10.0.1.0` through `10.0.1.3`).
        
- When an interface matched by this statement is found, two actions occur:
    1. EIGRP begins sending Hello packets out of that interface to discover neighbors.
    2. The local subnet connected to that interface is advertised to existing EIGRP neighbors.

##### `passive-interface vlan10`

- Prevents EIGRP from sending outbound routing updates and Hello packets on the specified interface (`Vlan10`).
    
- While it stops control traffic from leaving the interface, it **continues to advertise the subnet** attached to `Vlan10` to other EIGRP neighbors.
    
- This enhances network security and eliminates unnecessary CPU/bandwidth overhead by ensuring Hello packets are not broadcast onto end-user client segments where no EIGRP neighbors exist.

## Static Default Route Configuration

_(Be sure to do this step after you configure the `ISP-R1` IP address on it's `Gi0/0` interface then apply this configuration)._
### Commands to Run on ISP-EDGE1

```cli
ISP-EDGE1# configure terminal
ISP-EDGE1(config)#ip route 0.0.0.0 0.0.0.0 gigabitEthernet 0/2 10.10.1.2
ISP-EDGE1(config)#router eigrp 100
ISP-EDGE1(config-router)#redistribute static 
ISP-EDGE1(config-router)#end
```

### Command Syntax Explanation

##### `ip route 0.0.0.0 0.0.0.0 gigabitEthernet 0/2 10.10.1.2`

- Configures a static **default route** (often called the "gateway of last resort") on the router.
    
- The `0.0.0.0 0.0.0.0` address and mask combination acts as a wildcard match for **any network destination** that does not already exist in the local IP routing table.
    
##### `redistribute static`

- Takes all static routes configured on this router—specifically our `0.0.0.0/0` default route—and imports them into the EIGRP routing process.
    
- Automatically advertises this default route downstream to `EDGE1`, `EDGE2`, `CORE1`, and `CORE2`, allowing all internal switches to reach external networks without needing manual route entries on each device.
    
### Verification

Once applied, run this command on **`CORE1`** to verify that EIGRP received the default route:

```cli
CORE1# show ip route eigrp | include 0.0.0.0
```

You should see an entry starting with **`D*EX 0.0.0.0/0`** pointing toward `EDGE1`.

## Verification & Operational Health Checks

Execute the following commands on `ISP-EDGE1` and core devices to validate neighbor adjacencies and routing table health:

### 1. Verify Neighbor Adjacencies

Confirms that EIGRP hello exchange was successful and a two-way neighbor relationship is active.

```cli
ISP-EDGE1# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address         Interface       Hold Uptime   SRTT   RTO  Q  Seq
                                    (sec)         (ms)       Cnt Num
1   10.1.2.2        Gi2               12 00:14:22   12   200  0   14
0   10.1.1.2        Gi1               13 00:14:35   10   200  0   18
```

**Key Parameters to Check:**

- **Address:** The IP address of the remote neighbor interface.
    
- **Hold (sec):** The countdown timer for receiving the next Hello packet (defaults to 15s; resets upon arrival).
    
- **Q Cnt (Queue Count):** Should ideally be **`0`**. A value higher than zero indicates packet congestion or dropped routing updates.
    
- **SRTT (Smooth Round Trip Time):** The milliseconds taken for an EIGRP packet to reach the neighbor and receive an acknowledgment.
    

### 2. EIGRP Routing Table Inspection

Validates which routes have been learned dynamically via EIGRP.

```cli
CORE1# show ip route eigrp
Gateway of last resort is not set

      10.0.0.0/8 is subnetted, 5 subnets
D        10.0.2.0/30 [90/307200] via 10.0.1.2, 00:12:40, GigabitEthernet0/0
D        10.0.3.0/30 [90/307200] via 10.0.1.2, 00:12:40, GigabitEthernet0/0
D        10.1.1.0/30 [90/307200] via 10.0.1.2, 00:12:40, GigabitEthernet0/0
D        10.1.2.0/30 [90/358400] via 10.0.1.2, 00:11:15, GigabitEthernet0/0
```

**Output Decoding:**

- **`D`**: Indicates a route learned via internal EIGRP.
    
- **`[90/307200]`**:
    
    - `90` = Administrative Distance (AD) for Internal EIGRP (trustworthiness rating).
        
    - `307200` = Composite EIGRP Metric (calculated using bandwidth and delay by default).
        
- **`via 10.0.1.2`**: The next-hop IP address toward the destination.

# ISP/WAN Core OSPF Configuration

## Overview

Establishes multi-area OSPF across the WAN provider network:

- **Area 1 (Orange):** Connects `ISP-EDGE1` to `ISP-R1` and `ISP-R2`.
    
- **Area 0 Backbone (Green):** Core mesh connecting ABRs `ISP-R2` and `ISP-R6` via `ISP-R3`, `ISP-R4`, `ISP-R5`, and the central direct link.
    
- **Area 2 (Purple):** Connects `ISP-R6` to `ISP-R7` and `ISP-EDGE2`.
    
## WAN Interface & Subnet Allocation Table

### Area 1 (Orange) — WAN Subnet Table

|**Link Segment**|**Subnet**|**Local Device & Interface**|**Remote Device & Interface**|
|---|---|---|---|
|**ISP-EDGE1 $\leftrightarrow$ ISP-R1**|`10.10.1.0/30`|`ISP-EDGE1` (Gi0/2) — `10.10.1.1`|`ISP-R1` (f0/0) — `10.10.1.2`|
|**ISP-R1 $\leftrightarrow$ ISP-R2**|`10.10.1.4/30`|`ISP-R1` (f0/1) — `10.10.1.5`|`ISP-R2` (f0/0) — `10.10.1.6`|

### Area 0 Backbone (Green) — WAN Subnet Table

| **Link Segment**                             | **Subnet**      | **Local Device & Interface**   | **Remote Device & Interface**  |
| -------------------------------------------- | --------------- | ------------------------------ | ------------------------------ |
| **ISP-R2 $\leftrightarrow$ ISP-R3**          | `10.10.0.0/30`  | `ISP-R2` (f1/0) — `10.10.0.1`  | `ISP-R3` (f0/0) — `10.10.0.2`  |
| **ISP-R2 $\leftrightarrow$ ISP-R4**          | `10.10.0.4/30`  | `ISP-R2` (f2/0) — `10.10.0.5`  | `ISP-R4` (f0/0) — `10.10.0.6`  |
| **ISP-R2 $\leftrightarrow$ ISP-R5**          | `10.10.0.8/30`  | `ISP-R2` (f3/0) — `10.10.0.9`  | `ISP-R5` (f0/0) — `10.10.0.10` |
| **ISP-R3 $\leftrightarrow$ ISP-R6**          | `10.10.0.12/30` | `ISP-R3` (f0/1) — `10.10.0.13` | `ISP-R6` (f1/0) — `10.10.0.14` |
| **ISP-R4 $\leftrightarrow$ ISP-R6**          | `10.10.0.16/30` | `ISP-R4` (f0/1) — `10.10.0.17` | `ISP-R6` (f2/0) — `10.10.0.18` |
| **ISP-R5 $\leftrightarrow$ ISP-R6**          | `10.10.0.20/30` | `ISP-R5` (f0/1) — `10.10.0.21` | `ISP-R6` (f3/0) — `10.10.0.22` |

### Area 2 (Purple) — WAN Subnet Table

|**Link Segment**|**Subnet**|**Local Device & Interface**|**Remote Device & Interface**|
|---|---|---|---|
|**ISP-R6 $\leftrightarrow$ ISP-R7**|`10.10.2.0/30`|`ISP-R6` (f0/0) — `10.10.2.1`|`ISP-R7` (f0/0) — `10.10.2.2`|
|**ISP-R7 $\leftrightarrow$ ISP-EDGE2**|`10.10.2.4/30`|`ISP-R7` (f0/1) — `10.10.2.5`|`ISP-EDGE2` (Gi0/1) — `10.10.2.6`|

## Command Explanations

##### `router ospf 1`

- Initializes the OSPF routing process with Process ID `1` (locally significant).
    
##### `router-id X.X.X.X`

- Uniquely identifies the router within the OSPF domain using a 32-bit dotted-decimal number. Every router must have a unique ID to prevent database sync conflicts.
    
##### `network <IP> <Wildcard> area <ID>`

- Enables OSPF on matching interfaces and places them into the designated area (Area 0, 1, or 2).
    
## Device Configuration Steps

### 1. ISP-EDGE1 (Area 1)


```cli
ISP-EDGE1# configure terminal
ISP-EDGE1(config)#interface gigabitEthernet 0/2
ISP-EDGE1(config-if)#description Transit Link to ISP-R1 (f0/0)
ISP-EDGE1(config-if)#ip address 10.10.1.1 255.255.255.252
ISP-EDGE1(config-if)#ip address 10.10.1.1 255.255.255.252
ISP-EDGE1(config-if)#exit

ISP-EDGE1(config)#router ospf 1
ISP-EDGE1(config-router)#router-id 1.1.1.1
ISP-EDGE1(config-router)#network 10.10.1.0 0.0.0.3 area 1
ISP-EDGE1(config-router)#end

```

### 2. ISP-R1 (Area 1)

```cli
ISP-R1# configure terminal
ISP-R1(config)#interface fastEthernet 0/0
ISP-R1(config-if)#description Link to ISP-EDGE1 (Gi0/2)
ISP-R1(config-if)#ip address 10.10.1.2 255.255.255.252    
ISP-R1(config-if)#no shutdown 

ISP-R1(config)#interface fastEthernet 0/1
ISP-R1(config-if)#description Link to ISP-R2 (f0/0)
ISP-R1(config-if)#ip address 10.10.1.5 255.255.255.252 
ISP-R1(config-if)#no shutdown 

ISP-R1(config)#router ospf 1
ISP-R1(config-router)#router-id 10.1.1.1
ISP-R1(config-router)#network 10.10.1.0 0.0.0.3 area 1
ISP-R1(config-router)#network 10.10.1.4 0.0.0.3 area 1 
ISP-R1(config-router)#end
```

### 3. ISP-R2 (ABR: Area 1 & Area 0)

```
ISP-R2# configure terminal
ISP-R2(config)#interface fastEthernet 0/0
ISP-R2(config-if)#description Area 1 Link to ISP-R1 (f0/1)
ISP-R2(config-if)#ip address 10.10.1.6 255.255.255.252
ISP-R2(config-if)#no shutdown 

ISP-R2(config)#interface fastEthernet 1/0
ISP-R2(config-if)#description Area 0 Link to ISP-R3 (f0/0)
ISP-R2(config-if)#ip address 10.10.0.1 255.255.255.252
ISP-R2(config-if)#no shutdown 

ISP-R2(config)#interface fastEthernet 2/0
ISP-R2(config-if)#description Area 0 Link to ISP-R4 (f0/0)
ISP-R2(config-if)#ip address 10.10.0.5 255.255.255.252
ISP-R2(config-if)#no shutdown 

ISP-R2(config)#interface fastEthernet 3/0
ISP-R2(config-if)#description Area 0 Link to ISP-R5 (f0/0)
ISP-R2(config-if)#ip address 10.10.0.9 255.255.255.252
ISP-R2(config-if)#no shutdown 
ISP-R2(config-if)#exit 

ISP-R2(config)#router ospf 1
ISP-R2(config-router)#router-id 10.2.2.2
ISP-R2(config-router)#network 10.10.1.4 0.0.0.3 area 1
ISP-R2(config-router)#network 10.10.0.0 0.0.0.3 area 0 
ISP-R2(config-router)#network 10.10.0.4 0.0.0.3 area 0
ISP-R2(config-router)#network 10.10.0.8 0.0.0.3 area 0
ISP-R2(config-router)#end
```

### 4. ISP-R3 (Area 0 Backbone)

```cli
ISP-R3# configure terminal
ISP-R3(config)#interface fastEthernet 0/0

ISP-R3(config-if)#description Link to ISP-R2 (f1/0)
ISP-R3(config-if)#ip address 10.10.0.2 255.255.255.252
ISP-R3(config-if)#no shutdown 

ISP-R3(config)#interface fastEthernet 0/1
ISP-R3(config-if)#description Link to ISP-R6 (f1/0)
ISP-R3(config-if)#ip address 10.10.0.13 255.255.255.252
ISP-R3(config-if)#no shutdown 

ISP-R3(config)#router ospf 1
ISP-R3(config-router)#router-id 10.3.3.3
ISP-R3(config-router)#network 10.10.0.0 0.0.0.3 area 0
ISP-R3(config-router)#network 10.10.0.12 0.0.0.3 area 0
ISP-R3(config-router)#end
```

### 5. ISP-R4 (Area 0 Backbone)


```cli
ISP-R4# configure terminal
ISP-R4(config)#interface fastEthernet 0/0
ISP-R4(config-if)#description Link to ISP-R2 (f2/0)
ISP-R4(config-if)#ip address 10.10.0.6 255.255.255.252
ISP-R4(config-if)#no shutdown 

ISP-R4(config)#interface fastEthernet 0/1
ISP-R4(config-if)#description Link to ISP-R6 (f2/0)
ISP-R4(config-if)#ip address 10.10.0.17 255.255.255.252   
ISP-R4(config-if)#no shutdown 

ISP-R4(config)#router ospf 1
ISP-R4(config-router)#router-id 10.4.4.4
ISP-R4(config-router)#network 10.10.0.4 0.0.0.3 area 0
ISP-R4(config-router)#network 10.10.0.16 0.0.0.3 area 0
ISP-R4(config-router)#end
```

### 6. ISP-R5 (Area 0 Backbone)


```cli
ISP-R5# configure terminal
ISP-R5(config)#interface fastEthernet 0/0
ISP-R5(config-if)#description Link to ISP-R2 (f3/0)
ISP-R5(config-if)#ip address 10.10.0.10 255.255.255.252
ISP-R5(config-if)#no shutdown 

ISP-R5(config)#interface fastEthernet 0/1
ISP-R5(config-if)#description Link to ISP-R6 (f3/0)
ISP-R5(config-if)#ip address 10.10.0.21 255.255.255.252
ISP-R5(config-if)#no shutdown 

ISP-R5(config)#router ospf 1
ISP-R5(config-router)#router-id 10.5.5.5
ISP-R5(config-router)#network 10.10.0.8 0.0.0.3 area 0
ISP-R5(config-router)#network 10.10.0.20 0.0.0.3 area 0
ISP-R5(config-router)#end
```

### 7. ISP-R6 (ABR: Area 0 & Area 2)


```cli
ISP-R6# configure terminal
ISP-R6(config)#interface fastEthernet 1/0
ISP-R6(config-if)#description Area 0 Link to ISP-R3 (f0/1)
ISP-R6(config-if)#ip address 10.10.0.14 255.255.255.252
ISP-R6(config-if)#no shutdown 

ISP-R6(config)#interface fastEthernet 2/0
ISP-R6(config-if)#description Area 0 Link to ISP-R4 (f0/1)
ISP-R6(config-if)#ip address 10.10.0.18 255.255.255.252
ISP-R6(config-if)#no shutdown 

ISP-R6(config)#interface fastEthernet 3/0
ISP-R6(config-if)#description Area 0 Link to ISP-R5 (f0/1)
ISP-R6(config-if)#ip address 10.10.0.22 255.255.255.252
ISP-R6(config-if)#no shutdown 

ISP-R6(config)#interface fastEthernet 0/0
ISP-R6(config-if)#description Area 2 Link to ISP-R7 (f0/0)
ISP-R6(config-if)#ip address 10.10.2.1 255.255.255.252
ISP-R6(config-if)#no shutdown 

ISP-R6(config)#router ospf 1
ISP-R6(config-router)#router-id 10.6.6.6
ISP-R6(config-router)#network 10.10.0.12 0.0.0.3 are
ISP-R6(config-router)#network 10.10.0.16 0.0.0.3 area 0
ISP-R6(config-router)#network 10.10.0.20 0.0.0.3 area 0
ISP-R6(config-router)#network 10.10.2.0 0.0.0.3 area 2 
ISP-R6(config-router)#end
```

### 8. ISP-R7 (Area 2)


```cli
ISP-R7# configure terminal
ISP-R7(config)#interface fastEthernet 0/0
ISP-R7(config-if)#description Link to ISP-R6 (f0/0)
ISP-R7(config-if)#ip address 10.10.2.2 255.255.255.252
ISP-R7(config-if)#no shutdown 

ISP-R7(config)#interface fastEthernet 0/1          
ISP-R7(config-if)#description Link to ISP-EDGE2 (Gi0/1)
ISP-R7(config-if)#ip address 10.10.2.5 255.255.255.252
ISP-R7(config-if)#no shutdown 
ISP-R7(config-if)#exit 

ISP-R7(config)#router ospf 1
ISP-R7(config-router)#router-id 10.7.7.7 
ISP-R7(config-router)#network 10.10.2.0 0.0.0.3 area 2
ISP-R7(config-router)#network 10.10.2.4 0.0.0.3 area 2
ISP-R7(config-router)#end
```

### 9. ISP-EDGE2 (Area 2)


```cli
ISP-EDGE2# configure terminal
ISP-EDGE2(config)#interface gigabitEthernet 0/1
ISP-EDGE2(config-if)#description Link to ISP-R7 (f0/1)
ISP-EDGE2(config-if)#ip address 10.10.2.6 255.255.255.252
ISP-EDGE2(config-if)#no shutdown 
ISP-EDGE2(config-if)#exit 

ISP-EDGE2(config)#router ospf 1
ISP-EDGE2(config-router)#router-id 10.8.8.8
ISP-EDGE2(config-router)#network 10.10.2.4 0.0.0.3 area 2
ISP-EDGE2(config-router)#end
```

### Verification

We're going to run on **`ISP-EDGE1`** to check end-to-end OSPF route propagation through the WAN:

```cli
ISP-EDGE1# show ip ospf neighbor
ISP-EDGE1# show ip route ospf
```

You should see routes for Area 1 (`O`), Area 0 inter-area routes (`O IA`), and Area 2 inter-area routes (`O IA`) reaching all the way to `ISP-EDGE2` (`10.10.2.4/30`).

---

# NAT/PAT Configuration

To configure NAT/PAT overload on `ISP-EDGE1` while bypassing (exempting) traffic destined for the Remote Branch via the IPsec VPN, perform the following steps:

### 1. Define Campus Subnet Object Group

Combine all four HQ VLAN subnets into a single network object group. This keeps the configuration modular, easy to read, and simpler to manage if subnets are added later.

```cli
ISP-EDGE1# configure terminal

! Group internal HQ campus VLANs into a single object
ISP-EDGE1(config)# object-group network CAMPUS_VLANS
ISP-EDGE1(config-network-group)# 192.168.10.0 255.255.255.0
ISP-EDGE1(config-network-group)# 192.168.20.0 255.255.255.0
ISP-EDGE1(config-network-group)# 192.168.30.0 255.255.255.0
ISP-EDGE1(config-network-group)# 192.168.99.0 255.255.255.0
ISP-EDGE1(config-network-group)# exit
```


### 2. Configure the NAT Exemption Access List

Create an extended Access Control List referencing the `CAMPUS_VLANS` object group. The `deny` statement bypasses NAT translation for VPN traffic, while the `permit` statement enables PAT for outbound Internet traffic.

```cli
! Create NAT ACL using the object-group
ISP-EDGE1(config)# ip access-list extended NAT_ACL

! Deny branch-to-branch VPN traffic so it isn't translated (replace 172.16.0.0 0.0.255.255 with your actual Branch subnet)
ISP-EDGE1(config-ext-nacl)# deny ip object-group CAMPUS_VLANS 172.16.1.0 0.0.0.255

! Permit internal campus traffic heading to any other external address
ISP-EDGE1(config-ext-nacl)# permit ip object-group CAMPUS_VLANS any
ISP-EDGE1(config-ext-nacl)# exit
```

### 3. Enable NAT Overload (PAT)

Link the ACL to your outside WAN interface (`GigabitEthernet 0/2` facing `ISP-R1` or your public exit) using the `overload` keyword.

```cli
ISP-EDGE1(config)# ip nat inside source list NAT_ACL interface GigabitEthernet0/2 overload
```

### 4. Assign NAT Inside and Outside Interfaces

Designate which interfaces face your internal campus (LAN) and which face the ISP WAN.

```cli
! LAN Interface facing EDGE1
ISP-EDGE1(config)# interface GigabitEthernet0/0
ISP-EDGE1(config-if)# ip nat inside
ISP-EDGE1(config-if)# exit

! Second LAN Interface facing EDGE2 (Redundant Path)
ISP-EDGE1(config)# interface GigabitEthernet0/1
ISP-EDGE1(config-if)# ip nat inside
ISP-EDGE1(config-if)# exit

! WAN Interface facing ISP / OSPF Core
ISP-EDGE1(config)# interface GigabitEthernet0/2
ISP-EDGE1(config-if)# ip nat outside
ISP-EDGE1(config-if)# end
```

### 4. Verification Commands

- **Check active NAT translations:**

    ```
    ISP-EDGE1# show ip nat translations
    ```
    
    _(Confirms internal source IPs like `192.168.10.x` translate to `10.10.1.1` with allocated port numbers for public destinations)._
    
      
    
- **Verify translation statistics and ACL matches:**
    
    
    ```
    ISP-EDGE1# show ip nat statistics
    ```
    
- **Verify ACL & Object-Group Matches:**
    
    ```
    ISP-EDGE1# show access-lists NAT_ACL
    ```
    
    _(Traffic matching the `deny` rule bypasses NAT and retains its private source IP for the IPsec tunnel)._

# Remote Branch Network Configuration

## Overview

Establishes the lean Remote Branch infrastructure, connecting `BRANCH-PC1` through access switch `BRANCH-SW1` to gateway router `BRANCH-R1`, which uplink directly to the WAN edge router `ISP-EDGE2`.

  

## Interface & IP Allocation Table

| **Link Segment**        | **Subnet**      | **Local Device & Interface**           | **Remote Device & Interface**           | **Purpose**                  |
| ----------------------- | --------------- | -------------------------------------- | --------------------------------------- | ---------------------------- |
| **Branch Transit Link** | `10.10.3.0/30`  | `ISP-EDGE2` (`Gi0/0`) — `10.10.3.1/30` | `BRANCH-R1` (`f0/0`) — `10.10.3.2/30`   | Transit WAN Link to ISP Edge |
| **Branch LAN**          | `172.16.1.0/24` | `BRANCH-R1` (`f0/1`) — `172.16.1.1/24` | `BRANCH-SW1` (`e0/1`) — Unmanaged/Trunk | Remote Branch Gateway        |
| **Branch PC**           | `172.16.1.0/24` | `BRANCH-SW1` (`e0/0`) — Access         | `BRANCH-PC1` (`e0`) — `172.16.1.10/24`  | Remote LAN Host              |

## Device Configuration Steps

### 1. ISP-EDGE2

```cli
BRANCH-R1# configure terminal
! Configure tansit Interface facing BRANCH-R1

ISP-EDGE2(config)#interface gigabitEthernet 0/0
ISP-EDGE2(config-if)#ip address 10.10.3.1 255.255.255.252
ISP-EDGE2(config-if)#no shutdown 
ISP-EDGE2(config-if)#exit 

! Static Default Route pointing toward BRANCH-R1
ISP-EDGE2(config)#ip route 172.16.1.0 255.255.255.0 GigabitEthernet0/0 10.10.3.2
```

### 2. BRANCH-R1 (Gateway Router)

```cli
BRANCH-R1# configure terminal

! Configure tansit Interface facing ISP-EDGE2
BRANCH-R1(config)# interface FastEthernet0/0
BRANCH-R1(config-if)# description Link to ISP-EDGE2 (Gi0/0)
BRANCH-R1(config-if)# ip address 10.10.3.2 255.255.255.252
BRANCH-R1(config-if)# no shutdown
BRANCH-R1(config-if)# exit

! Configure LAN Interface facing BRANCH-SW1
BRANCH-R1(config)# interface FastEthernet0/1
BRANCH-R1(config-if)# description Gateway
BRANCH-R1(config-if)# ip address 172.16.1.1 255.255.255.0
BRANCH-R1(config-if)# no shutdown
BRANCH-R1(config-if)# exit

! Static Default Route pointing toward ISP-EDGE2
BRANCH-R1(config)# ip route 0.0.0.0 0.0.0.0 FastEthernet0/0 10.10.3.1
BRANCH-R1(config)# end
BRANCH-R1# write memory
```

### 3. BRANCH-SW1 (Branch Access Switch)

```cli
BRANCH-SW1# configure terminal

! Changing STP Mode
BRANCH-SW1(config)#spanning-tree mode rapid-pvst 

! Configure Uplink to BRANCH-R1
BRANCH-SW1(config)# interface ethernet 0/1
BRANCH-SW1(config-if)# description Uplink to BRANCH-R1 (f0/1)
BRANCH-SW1(config-if)#switchport trunk encapsulation dot1q 
BRANCH-SW1(config-if)#switchport mode trunk 
BRANCH-SW1(config-if)# exit

! Configure Access Port for BRANCH-PC1
BRANCH-SW1(config)# interface ethernet 0/0
BRANCH-SW1(config-if)# description Access Port for BRANCH-PC1
BRANCH-SW1(config-if)# switchport mode access
BRANCH-SW1(config-if)#spanning-tree portfast edge 
BRANCH-SW1(config-if)# end
```

### 3. BRANCH-PC1 (VPCS Configuration)


```cli
VPCS> set pcname BR-PC1
BR-PC1> ip 172.16.1.10 255.255.255.0 172.16.1.1
BR-PC1> save
```

### 4. Verification Commands

- **From BRANCH-PC1:**
    
    ```cli
    BRANCH-PC1> ping 172.16.1.1
    BRANCH-PC1> ping 10.10.3.1
    ```
    
# Site-to-Site IPsec VPN Configuration

Step-by-step implementation guide to build the **IKEv1 / IPsec Site-to-Site VPN Tunnel** between **`ISP-EDGE1`** (HQ Edge) and **`ISP-EDGE2`** (Remote Edge).
## Overview

Establishes an encrypted IKEv1 / IPsec Site-to-Site VPN tunnel across the ISP network between `ISP-EDGE1` and `ISP-EDGE2` to securely interconnect HQ VLANs (`192.168.0.0/16` space) with the Remote Branch subnet (`172.16.1.0/24`).

## VPN Parameter Summary

|**Parameter**|**Phase 1 (ISAKMP)**|**Phase 2 (IPsec)**|
|---|---|---|
|**Authentication**|Pre-Shared Key (`cisco123`)|ESP Protocol|
|**Encryption**|AES-256|AES-256|
|**Hash / Integrity**|SHA-256|SHA-HMAC|
|**Diffie-Hellman Group**|Group 14 (2048-bit)|Group 14 (PFS)|
|**Key Exchange Lifetime**|86400 seconds (24 hrs)|3600 seconds (1 hr)|

## Device Configuration Steps

### 1. ISP-EDGE1 Configuration (HQ Gateway)


```cli
ISP-EDGE1# configure terminal

! Step 1: Define Phase 1 ISAKMP Policy
ISP-EDGE1(config)# crypto isakmp policy 10
ISP-EDGE1(config-isakmp)# encryption aes 256
ISP-EDGE1(config-isakmp)# hash sha256
ISP-EDGE1(config-isakmp)# authentication pre-share
ISP-EDGE1(config-isakmp)# group 14
ISP-EDGE1(config-isakmp)# lifetime 86400
ISP-EDGE1(config-isakmp)# exit

! Step 2: Define Pre-Shared Key for Remote Peer (ISP-EDGE2 WAN IP)
ISP-EDGE1(config)# crypto isakmp key cisco123 address 10.10.2.6

! Step 3: Define Phase 2 Transform Set
ISP-EDGE1(config)# crypto ipsec transform-set TS-AES256-SHA esp-aes 256 esp-sha-hmac
ISP-EDGE1(cfg-crypto-trans)# mode tunnel
ISP-EDGE1(cfg-crypto-trans)# exit

! Step 4: Define Interesting Traffic ACL (HQ VLANs -> Remote LAN)
ISP-EDGE1(config)# ip access-list extended VPN_TRAFFIC_ACL
ISP-EDGE1(config-ext-nacl)# permit ip object-group CAMPUS_VLANS 172.16.1.0 0.0.0.255
ISP-EDGE1(config-ext-nacl)# exit

! Step 5: Define Crypto Map
ISP-EDGE1(config)# crypto map HQ_TO_BRANCH_MAP 10 ipsec-isakmp
ISP-EDGE1(config-crypto-map)# set peer 10.10.2.6
ISP-EDGE1(config-crypto-map)# set transform-set TS-AES256-SHA
ISP-EDGE1(config-crypto-map)# set pfs group14
ISP-EDGE1(config-crypto-map)# match address VPN_TRAFFIC_ACL
ISP-EDGE1(config-crypto-map)# exit

! Step 6: Apply Crypto Map to Outside WAN Interface
ISP-EDGE1(config)# interface gigabitethernet 0/2
ISP-EDGE1(config-if)# crypto map HQ_TO_BRANCH_MAP
ISP-EDGE1(config-if)# end

! Step 7: Add Static Route to steer Remote Branch traffic into the Crypto Map ISP-EDGE1(config)# ip route 172.16.1.0 255.255.255.0 gigabitethernet 0/2 10.10.1.2 ISP-EDGE1(config)# end
ISP-EDGE1# write memory
```

### 2. ISP-EDGE2 Configuration (Branch Edge Gateway)


```cli
ISP-EDGE2# configure terminal

! Step 1: Define Phase 1 ISAKMP Policy
ISP-EDGE2(config)# crypto isakmp policy 10
ISP-EDGE2(config-isakmp)# encryption aes 256
ISP-EDGE2(config-isakmp)# hash sha256
ISP-EDGE2(config-isakmp)# authentication pre-share
ISP-EDGE2(config-isakmp)# group 14
ISP-EDGE2(config-isakmp)# lifetime 86400
ISP-EDGE2(config-isakmp)# exit

! Step 2: Define Pre-Shared Key for HQ Peer (ISP-EDGE1 WAN IP)
ISP-EDGE2(config)# crypto isakmp key cisco123 address 10.10.1.1

! Step 3: Define Phase 2 Transform Set
ISP-EDGE2(config)# crypto ipsec transform-set TS-AES256-SHA esp-aes 256 esp-sha-hmac
ISP-EDGE2(cfg-crypto-trans)# mode tunnel
ISP-EDGE2(cfg-crypto-trans)# exit

ISP-EDGE2# configure terminal

! Step 4: Define HQ Campus Subnets Object Group
ISP-EDGE2(config)# object-group network HQ_CAMPUS_VLANS
ISP-EDGE2(config-network-group)# 192.168.10.0 255.255.255.0
ISP-EDGE2(config-network-group)# 192.168.20.0 255.255.255.0
ISP-EDGE2(config-network-group)# 192.168.30.0 255.255.255.0
ISP-EDGE2(config-network-group)# 192.168.99.0 255.255.255.0
ISP-EDGE2(config-network-group)# exit

! Step 5: Define Interesting Traffic ACL (Remote LAN -> HQ VLANs)
ISP-EDGE2(config)# ip access-list extended REMOTE_VPN_TRAFFIC_ACL
ISP-EDGE2(config-ext-nacl)# permit ip 172.16.1.0 0.0.0.255 object-group HQ_CAMPUS_VLANS
ISP-EDGE2(config-ext-nacl)# exit

! Step 6: Define Crypto Map
ISP-EDGE2(config)# crypto map BRANCH_TO_HQ_MAP 10 ipsec-isakmp
ISP-EDGE2(config-crypto-map)# set peer 10.10.1.1
ISP-EDGE2(config-crypto-map)# set transform-set TS-AES256-SHA
ISP-EDGE2(config-crypto-map)# set pfs group14
ISP-EDGE2(config-crypto-map)# match address REMOTE_VPN_TRAFFIC_ACL
ISP-EDGE2(config-crypto-map)# exit

! Step 7: Apply Crypto Map to Outside WAN Interface
ISP-EDGE2(config)# interface gigabitethernet 0/1
ISP-EDGE2(config-if)# crypto map BRANCH_TO_HQ_MAP
ISP-EDGE2(config-if)# end
ISP-EDGE2# write memory

! Routes for HQ Campus VLANs into the VPN Crypto Map
ISP-EDGE2(config)#ip route 192.168.10.0 255.255.255.0 gigabitethernet 0/1 10.10.2.5
ISP-EDGE2(config)#ip route 192.168.20.0 255.255.255.0 gigabitethernet 0/1 10.10.2.5
ISP-EDGE2(config)#ip route 192.168.30.0 255.255.255.0 gigabitethernet 0/1 10.10.2.5
ISP-EDGE2(config)#ip route 192.168.99.0 255.255.255.0 gigabitethernet 0/1 10.10.2.5
```

## Tunnel Verification & Testing

### 1. Trigger the Tunnel

IPsec tunnels are brought up on-demand when "interesting traffic" matches the crypto ACL. Initiate traffic from `BR-PC1`:


```cli
BR-PC1> ping 192.168.10.128
```

### 2. Verify ISAKMP (Phase 1)

Check if the Phase 1 IKE Security Association is established (`QM_IDLE` status indicates success):


```cli
ISP-EDGE1# show crypto isakmp sa
```

_Expected Output:_


```cli
dst             src             state          conn-id status
10.10.2.6       10.10.1.1       QM_IDLE           1001 ACTIVE
```

### 3. Verify IPsec SA (Phase 2)

Verify active encrypted/decrypted packet counters:

```cli
ISP-EDGE1# show crypto ipsec sa
```

_Expected Output:_


```cli
#pkts encaps: 5, #pkts encrypt: 5, #pkts digest: 5
#pkts decaps: 5, #pkts decrypt: 5, #pkts verify: 5
```

Here is the complete configuration section for your HQ switch infrastructure covering SSH, Port Security, BPDU Guard, DHCP Snooping, and Dynamic ARP Inspection (DAI), formatted for your Obsidian documentation.

  

# HQ Access Switches - Layer 2 Security Hardening

## 1. Management & SSH Setup

Replaces insecure Telnet with encrypted SSH v2 management.

```cli

ISP-EDGE1# configure terminal

! 1. Define domain name for RSA key generation
ISP-EDGE1(config)# ip domain-name lab.local

! 2. Generate 2048-bit RSA key pair
ISP-EDGE1(config)# crypto key generate rsa general-keys modulus 2048

! 3. Enforce SSH Version 2
ISP-EDGE1(config)# ip ssh version 2

! 4. Create local administrative credentials
ISP-EDGE1(config)# username admin privilege 15 secret Cisco123!

! 5. Restrict VTY lines to SSH only
ISP-EDGE1(config)# line vty 0 15
ISP-EDGE1(config-line)# transport input ssh
ISP-EDGE1(config-line)# login local
ISP-EDGE1(config-line)# exit

ISP-EDGE1# write memory
```

### Verification Commands

Run these on `ISP-EDGE1` to verify operational status and monitor active management sessions:

```
! Check SSH server status and version
ISP-EDGE1# show ip ssh

! View active SSH connections and client IP addresses
ISP-EDGE1# show ssh
```

### Remote Access Test (Management)

> [!TIP] Connecting from the Management (`192.168.99.4`) To access `ISP-EDGE1` over the network, target its internal interface IP (e.g., `10.1.1.1` or `10.1.2.1` via routing from VLAN 99):
> 
> 
> ```bash
> osboxes@osboxes:~$ ssh admin@10.1.1.1
> ```

## 2. Port Security (Access Interfaces)

### Port Security Configuration

Prevents unauthorized devices from connecting and mitigates MAC flooding attacks on end-user access ports.

```cli
! Apply Port Security to user access interface range (e.g., Eth0/0 - 3)
ACCESS1(config)#interface range ethernet 0/0-1
ACCESS1(config-if-range)# switchport port-security
ACCESS1(config-if-range)# switchport port-security maximum 2
ACCESS1(config-if-range)# switchport port-security mac-address sticky
ACCESS1(config-if-range)# switchport port-security violation shutdown
ACCESS1(config-if-range)# exit
```

> [!NOTE] Port Security Violation Modes
> 
>   
> 
> - **`restrict`:** Drops unauthorized frames, generates a syslog/SNMP alert, and increments the violation counter without taking the port down.
>     
>       
>     
> - **`shutdown`:** Immediately drops traffic and places the interface into an `err-disabled` state upon violation.
>     
>       
> - **`mac-address sticky:`** Dynamically learns connected MAC addresses and saves them into the running configuration.
>   
>   
>   
> - **`maximum 2:`** Allows up to 2 distinct MAC addresses per physical port before triggering a violation.

> [!TIP] Implement this configuration on all access switches edge ports.

### Verification & Troubleshooting Commands

Run these on access switches to inspect port security status or clear violations:

```cli
! View global port security summary and total violations
ACCESS1# show port-security

! View port security configuration and learned MACs on a specific interface
ACCESS1# show port-security interface ethernet 0/0

! View all dynamically learned sticky MAC addresses
ACCESS1# show port-security address

! Check if any interfaces are currently in err-disabled state
ACCESS1# show interface status err-disabled
```

> [!TIP] Manual Recovery from `err-disabled` State If an interface enters `err-disable` due to a port security violation, you can manually reset it:

```cli
ACCESS1(config)# interface ethernet 0/0
ACCESS1(config-if)# shutdown
ACCESS1(config-if)# no shutdown
```

## 3. Spanning Tree BPDU Guard

**BPDU Guard** is a Cisco Layer 2 security feature that protects Spanning Tree Protocol (STP) topology by instantly disabling a switch port if it receives an incoming Bridge Protocol Data Unit (BPDU) frame.

```cli
! Enable STP PortFast and BPDU Guard globally on all access interfaces
ACCESS1# configure terminal
ACCESS1(config)# spanning-tree portfast bpduguard default

! Apply individually to specific interface ranges
ACCESS1(config)# interface range ethernet 0/0-1
ACCESS1(config-if-range)# spanning-tree bpduguard enable
ACCESS1(config-if-range)# exit

! Optional: Auto-recover interfaces placed into err-disabled state by BPDU Guard
ACCESS1(config)# errdisable recovery cause bpduguard
ACCESS1(config)# errdisable recovery interval 300
ACCESS1(config)# exit
ACCESS1# write memory
```

###  Breakdown

| **Feature**       | **Scope**    | **Function**                                                                                                  |
| ----------------- | ------------ | ------------------------------------------------------------------------------------------------------------- |
| **PortFast**      | Access Ports | Bypasses the listening and learning states, transitioning the port immediately into the forwarding state.     |
| **BPDU Guard**    | Edge Ports   | Automatically disables (`err-disabled`) an interface if a Bridge Protocol Data Unit (BPDU) is received on it. |
| **Auto-Recovery** | Global       | Automatically restores a `bpduguard` err-disabled interface after a 300-second (5-minute) timer.              |

### Verification Commands

Run these on access switches to verify STP edge port status and monitor for violations:

```cli
! Check global PortFast and BPDU Guard status
ACCESS1# show spanning-tree summary

! Inspect BPDU Guard operational status on a specific port
ACCESS1# show spanning-tree interface ethernet 0/0 detail

! Verify if any ports are down due to receiving rogue BPDUs
ACCESS1# show interface status err-disabled
```

> [!WARNING] BPDU Guard Operational Rule
> 
> BPDU Guard should **only** be deployed on end-user access ports (PCs, printers, IP phones). Never enable BPDU Guard on trunk links or inter-switch links connecting to other switches or routers.
## 4. DHCP Snooping & Dynamic ARP Inspection (DAI)

Prevents rogue DHCP server attacks and defends against ARP poisoning/spoofing by validating incoming ARP packets against the DHCP Snooping binding database.

```cli
! 1. Enable DHCP Snooping globally and on production VLANs

ACCESS1# configure terminal
ACCESS1(config)# ip dhcp snooping
ACCESS1(config)# ip dhcp snooping vlan 10,20,30

! 2. Enable Dynamic ARP Inspection (DAI) on production VLANs

ACCESS1(config)# ip arp inspection vlan 10,20,30

! 3. Configure Trust on physical uplink/trunk interface range

ACCESS1(config)# interface range ethernet 0/2-3
ACCESS1(config-if-range)# ip dhcp snooping trust
ACCESS1(config-if-range)# ip arp inspection trust
ACCESS1(config-if-range)# exit

! 4. Configure Trust on EtherChannel links (Choose Option A or B)

! --- Option A: Apply directly to physical member interfaces ---

ACCESS1(config)# interface range ethernet 1/0-1
ACCESS1(config-if-range)# ip dhcp snooping trust
ACCESS1(config-if-range)# ip arp inspection trust
ACCESS1(config-if-range)# exit

! --- Option B: Apply to logical Port-Channel (It inherits down to member ports, Recommended to prevent %EC-5 syslog warnings) ---

ACCESS1(config)# interface port-channel 1
ACCESS1(config-if)# ip dhcp snooping trust
ACCESS1(config-if)# ip arp inspection trust
ACCESS1(config-if)# end

ACCESS1# write memory
```

> [!TIP] Implementation Sequence
> 
> Ensure **DHCP Snooping** is fully configured and hosts have acquired DHCP leases _before_ enabling **Dynamic ARP Inspection (DAI)**. DAI relies on the DHCP Snooping binding table to permit legitimate ARP traffic; enabling DAI beforehand will drop all ARP traffic from unmapped hosts.
### Security Feature Summary

|**Feature**|**Untrusted Ports (Access Ports)**|**Trusted Ports (Trunk Uplinks)**|**Key Function**|
|---|---|---|---|
|**DHCP Snooping**|Drops DHCP Server messages (`OFFER`, `ACK`)|Allows all DHCP traffic|Prevents rogue DHCP servers & builds the IP-to-MAC binding database.|
|**Dynamic ARP Inspection**|Validates ARP packets against the binding database|Bypasses ARP validation checks|Blocks Man-in-the-Middle (MitM) ARP poisoning and spoofing attacks.|

### Verification & Monitoring Commands

Run these on access switches to inspect binding database records and verify DAI status:

```cli
! Check global DHCP Snooping operational status and interface trust states
ACCESS1# show ip dhcp snooping

! View active DHCP Snooping IP-to-MAC binding database entries
ACCESS1# show ip dhcp snooping binding

! Inspect Dynamic ARP Inspection status and dropped packet statistics per VLAN
ACCESS1# show ip arp inspection vlan 10
ACCESS1#show ip arp inspection vlan 10,20,30

! View DAI statistics and execution counters
ACCESS1# show ip arp inspection statistics
```

> [!WARNING] Mandatory Interface Trusting
> 
> Trunk ports connected to core switches or legitimate DHCP servers **must** be explicitly configured as trusted for both features (`ip dhcp snooping trust` and `ip arp inspection trust`). Leaving an uplink untrusted will cause all valid DHCP offers and inter-switch ARP traffic to be dropped immediately.

