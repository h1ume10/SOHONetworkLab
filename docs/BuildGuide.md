# Build Guide

This guide rebuilds the SOHO network from an empty Packet Tracer workspace. Every command here matches the exported files in the `configs` folder. Passwords are shown as placeholders. Replace them with your own values.

## Contents

1. [Place and cable the devices](#stage-1-place-and-cable-the-devices)
2. [Configure the ISP router and WAN link](#stage-2-configure-the-isp-router-and-wan-link)
3. [Configure the switch, VLANs and trunk](#stage-3-configure-the-switch-vlans-and-trunk)
4. [Configure router gateways and DHCP](#stage-4-configure-router-gateways-and-dhcp)
5. [Add NAT for internet access](#stage-5-add-nat-for-internet-access)
6. [Add guest isolation](#stage-6-add-guest-isolation)
7. [Configure the wireless networks](#stage-7-configure-the-wireless-networks)
8. [Harden the router and switch](#stage-8-harden-the-router-and-switch)
9. [Verify and export](#stage-9-verify-and-export)

The design values used throughout are in the [Design Document](DesignDocument.md).

## Stage 1: Place and cable the devices

1. Open a new Packet Tracer file and place these devices, renaming each one:
   1. Two 2911 routers: `SOHO_R1` and `ISP_R1`
   2. One 2960 switch: `SOHO_SW1`
   3. Two access points: `AP_Staff` and `AP_Guest`
   4. Two PCs: `Staff_PC1` and `Staff_PC2`
   5. One printer: `Printer1`
   6. One smartphone: `Staff_Phone`
   7. One tablet: `Guest_Tablet`
   8. One server: `WebServer`
2. Connect them with copper cables:

| From | To | Cable |
| --- | --- | --- |
| SOHO_R1 Gi0/0 | SOHO_SW1 Gi0/1 | Straight through |
| SOHO_R1 Gi0/1 | ISP_R1 Gi0/0 | Crossover |
| ISP_R1 Gi0/1 | WebServer Fa0 | Crossover |
| SOHO_SW1 Fa0/1 | Staff_PC1 | Straight through |
| SOHO_SW1 Fa0/2 | Staff_PC2 | Straight through |
| SOHO_SW1 Fa0/3 | Printer1 | Straight through |
| SOHO_SW1 Fa0/4 | AP_Staff | Straight through |
| SOHO_SW1 Fa0/5 | AP_Guest | Straight through |

3. Leave the phone and tablet uncabled. They join over Wi-Fi in Stage 7.

Expect red or amber link lights on every router connection at this point. Router interfaces are switched off by default and turn green once they are configured.

## Stage 2: Configure the ISP router and WAN link

The ISP router simulates the provider. It gives the office a WAN link and hosts the web server that stands in for the internet.

1. On `ISP_R1`, open the CLI and run:

```
enable
configure terminal
hostname ISP_R1
interface gigabitEthernet 0/0
description Link to SOHO_R1
ip address 203.0.113.1 255.255.255.252
no shutdown
exit
interface gigabitEthernet 0/1
description Link to WebServer
ip address 198.51.100.1 255.255.255.0
no shutdown
exit
end
write memory
```

2. On `SOHO_R1`, open the CLI and run:

```
enable
configure terminal
hostname SOHO_R1
interface gigabitEthernet 0/1
description WAN to ISP_R1
ip address 203.0.113.2 255.255.255.252
no shutdown
exit
interface gigabitEthernet 0/0
description LAN trunk to SOHO_SW1
no shutdown
exit
ip route 0.0.0.0 0.0.0.0 203.0.113.1
end
write memory
```

The LAN interface has no address. The VLAN gateways go on subinterfaces in Stage 4.

3. On `WebServer`, open Desktop, then IP Configuration, choose Static and enter:
   1. IPv4 address: 198.51.100.10
   2. Subnet mask: 255.255.255.0
   3. Default gateway: 198.51.100.1

**Verify**

1. On `SOHO_R1`, run `ping 203.0.113.1` and `ping 198.51.100.10`. Both should reply. The first echo of each may time out while addresses are learned.

## Stage 3: Configure the switch, VLANs and trunk

1. On `SOHO_SW1`, open the CLI and run:

```
enable
configure terminal
hostname SOHO_SW1
vlan 10
name STAFF
vlan 20
name GUEST
vlan 30
name DEVICES
vlan 99
name MANAGEMENT
exit
interface range fastEthernet 0/1 - 2
description Staff PCs
switchport mode access
switchport access vlan 10
exit
interface fastEthernet 0/3
description Printer
switchport mode access
switchport access vlan 30
exit
interface fastEthernet 0/4
description AP Staff
switchport mode access
switchport access vlan 10
exit
interface fastEthernet 0/5
description AP Guest
switchport mode access
switchport access vlan 20
exit
interface gigabitEthernet 0/1
description Trunk to SOHO_R1
switchport mode trunk
switchport trunk native vlan 99
switchport trunk allowed vlan 10,20,30,99
exit
interface vlan 99
description Switch management
ip address 192.168.99.2 255.255.255.0
no shutdown
exit
ip default-gateway 192.168.99.1
end
write memory
```

2. The Guest access point sits in VLAN 20, so anything joining the guest wireless network lands in the guest VLAN.

**Verify**

1. Run `show vlan brief`. STAFF should hold Fa0/1, Fa0/2 and Fa0/4, GUEST should hold Fa0/5, and DEVICES should hold Fa0/3.
2. Run `show interfaces trunk`. Gi0/1 should be trunking with VLANs 10, 20, 30 and 99 allowed and native VLAN 99.
3. Run these on the switch itself. Running them on the router shows an empty table, because a router has no VLAN table.

## Stage 4: Configure router gateways and DHCP

The router reaches all four VLANs through the one trunk cable, using one subinterface per VLAN. This design is often called router on a stick.

1. On `SOHO_R1`, run:

```
enable
configure terminal
interface gigabitEthernet 0/0
no shutdown
exit
interface gigabitEthernet 0/0.10
description STAFF gateway
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
exit
interface gigabitEthernet 0/0.20
description GUEST gateway
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
exit
interface gigabitEthernet 0/0.30
description DEVICES gateway
encapsulation dot1Q 30
ip address 192.168.30.1 255.255.255.0
exit
interface gigabitEthernet 0/0.99
description MANAGEMENT gateway
encapsulation dot1Q 99 native
ip address 192.168.99.1 255.255.255.0
exit
ip dhcp excluded-address 192.168.10.1 192.168.10.20
ip dhcp excluded-address 192.168.20.1 192.168.20.20
ip dhcp excluded-address 192.168.30.1 192.168.30.20
ip dhcp pool STAFF
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 198.51.100.10
exit
ip dhcp pool GUEST
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
dns-server 198.51.100.10
exit
ip dhcp pool DEVICES
network 192.168.30.0 255.255.255.0
default-router 192.168.30.1
dns-server 198.51.100.10
exit
end
write memory
```

2. The `native` keyword on VLAN 99 must match the native VLAN set on the switch trunk. A mismatch between the two is a common real world fault.
3. Give the printer a fixed address. On `Printer1`, open Config, then FastEthernet0, choose Static and enter:
   1. IPv4 address: 192.168.30.10
   2. Subnet mask: 255.255.255.0
   3. Default gateway: 192.168.30.1

**Verify**

1. On `SOHO_R1`, run `show ip interface brief`. All four Gi0/0 subinterfaces should show their addresses and be up, and Gi0/1 should still show 203.0.113.2.
2. On `SOHO_R1`, run `ping 192.168.99.2` to reach the switch.
3. On `Staff_PC1` and `Staff_PC2`, open IP Configuration and choose DHCP. They should receive addresses beginning with 192.168.10, starting at .21, with gateway 192.168.10.1.
4. On `Staff_PC1`, open Command Prompt and run `ping 192.168.10.1` and `ping 192.168.30.10`.

Both pings should reply, though the first echo may time out.
