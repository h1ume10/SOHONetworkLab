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

## Stage 5: Add NAT for internet access

Staff and guests share the single public address on the WAN interface. The Devices network is left out on purpose, because a printer does not need the internet.

1. On `SOHO_R1`, run:

```
enable
configure terminal
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.20.0 0.0.0.255
ip nat inside source list 1 interface gigabitEthernet 0/1 overload
interface gigabitEthernet 0/1
ip nat outside
exit
interface gigabitEthernet 0/0.10
ip nat inside
exit
interface gigabitEthernet 0/0.20
ip nat inside
exit
end
write memory
```

**Verify**

1. On `Staff_PC1`, run `ping 198.51.100.10`, then open the Web Browser and load `198.51.100.10`. The server page should appear.
2. Straight afterwards, on `SOHO_R1`, run `show ip nat translations`. Entries expire within about a minute, so run it immediately. The Inside global column should show 203.0.113.2 and the Inside local column should show a 192.168.10 address.

## Stage 6: Add guest isolation

Guests may reach the internet but nothing inside the office. The access list is applied inbound on the guest subinterface, so blocked traffic is dropped at the first router interface it meets.

1. On `SOHO_R1`, run:

```
enable
configure terminal
ip access-list extended GUEST_IN
deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
deny ip 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255
deny ip 192.168.20.0 0.0.0.255 192.168.99.0 0.0.0.255
permit ip any any
exit
interface gigabitEthernet 0/0.20
ip access-group GUEST_IN in
end
write memory
```

The three deny lines protect the Staff, Devices and Management networks. The final permit line allows everything else, which is the internet. Without it, an access list ends with an invisible deny of all traffic.

**Verify**

Run these after the wireless networks exist in Stage 7, because they need the guest tablet.

1. On `SOHO_R1`, run `show access-lists`. GUEST_IN should list its four lines.
2. After the tablet tests in Stage 7, run `show access-lists` again. The deny lines should now show match counts.

## Stage 7: Configure the wireless networks

Each access point sits in its own VLAN, so the network name decides which VLAN a device lands in.

1. On `AP_Staff`, open Config, then Port 1:
   1. SSID: `SOHO_Staff`
   2. Authentication: WPA2 PSK
   3. Passphrase: choose your own, at least 12 characters
   4. Encryption: AES
2. On `AP_Guest`, open Config, then Port 1:
   1. SSID: `SOHO_Guest`
   2. Authentication: WPA2 PSK
   3. Passphrase: choose a different one from the staff passphrase
   4. Encryption: AES
3. Place `Staff_Phone` close to `AP_Staff` and `Guest_Tablet` close to `AP_Guest`.
4. On `Staff_Phone`, open Config, then Wireless0. Set the SSID to `SOHO_Staff`, choose WPA2 PSK, enter the staff passphrase and set IP configuration to DHCP.
5. On `Guest_Tablet`, open Config, then Wireless0. Set the SSID to `SOHO_Guest`, choose WPA2 PSK, enter the guest passphrase and set IP configuration to DHCP.
6. Use Fast Forward Time until a dotted wireless link appears for each device.

**Verify**

1. `Staff_Phone` should receive an address beginning with 192.168.10. From its Command Prompt, `ping 192.168.30.10` and `ping 198.51.100.10` should both reply.
2. `Guest_Tablet` should receive an address beginning with 192.168.20 and gateway 192.168.20.1. From its Command Prompt, run these four:
   1. `ping 198.51.100.10` should reply, because guests keep internet access.
   2. `ping 192.168.20.1` should reply, because it is the tablet's own gateway.
   3. `ping 192.168.30.10` should fail, because the printer is blocked.
   4. `ping 192.168.10.21` should fail, because staff devices are blocked.

The two failures are the proof that isolation works.

## Stage 8: Harden the router and switch

This stage adds encrypted credentials, SSH only remote access, a remote access list, a login banner and an idle timeout. The switch also closes its unused ports and limits each staff port with port security.

Replace `YOUR_ENABLE_SECRET` and `YOUR_ADMIN_PASSWORD` with your own values. Use a different value for each, and store them in a password manager. Key generation takes a few seconds, so run the first block, wait for the prompt to return, then run the second.

1. On `SOHO_R1`, run block A:

```
enable
configure terminal
no ip domain-lookup
service password-encryption
enable secret YOUR_ENABLE_SECRET
username sohoadmin secret YOUR_ADMIN_PASSWORD
ip domain-name soho.local
crypto key generate rsa general-keys modulus 2048
```

2. Then block B:

```
ip ssh version 2
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 permit 192.168.99.0 0.0.0.255
banner motd #Authorised access only#
line console 0
logging synchronous
login local
exit
line vty 0 4
access-class 10 in
login local
transport input ssh
exec-timeout 5 0
exit
end
write memory
```

3. On `SOHO_SW1`, run block A with the same values:

```
enable
configure terminal
no ip domain-lookup
service password-encryption
enable secret YOUR_ENABLE_SECRET
username sohoadmin secret YOUR_ADMIN_PASSWORD
ip domain-name soho.local
crypto key generate rsa general-keys modulus 2048
```

4. Then block B for the switch:

```
ip ssh version 2
access-list 10 permit 192.168.10.0 0.0.0.255
access-list 10 permit 192.168.99.0 0.0.0.255
banner motd #Authorised access only#
vlan 999
name UNUSED
exit
interface range fastEthernet 0/6 - 24 , gigabitEthernet 0/2
switchport mode access
switchport access vlan 999
shutdown
exit
interface range fastEthernet 0/1 - 2
switchport port-security
switchport port-security maximum 2
switchport port-security mac-address sticky
switchport port-security violation restrict
exit
line console 0
logging synchronous
login local
exit
line vty 0 15
access-class 10 in
login local
transport input ssh
exec-timeout 5 0
exit
end
write memory
```

**Verify**

1. On each device, run `show ip ssh`. It should report SSH enabled, version 2.0.
2. On `Staff_PC1`, run `ssh -l sohoadmin 192.168.10.1`. After the password, the router prompt should appear.
3. On `Guest_Tablet`, run `ssh -l sohoadmin 192.168.20.1`. It should fail without asking for a password. This failure is the evidence.
4. On `SOHO_SW1`, run `show interfaces status`. Fa0/6 and up should
