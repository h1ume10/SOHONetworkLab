# Design Document

## 1. Purpose

This document records the requirements, design decisions and addressing plan for the SOHO network. It is the reference for the build guide, the test plan and the exported configurations.

## 2. Scenario

A five person consulting office works from a converted garage. The office needs a staff network, a separate guest Wi-Fi network, a shared printer and one shared internet connection. The scenario is fictional.

## 3. Requirements

| ID | Requirement |
| --- | --- |
| R1 | Staff devices must reach each other, the printer and the internet |
| R2 | Guest devices must reach the internet only |
| R3 | Guests must not reach staff devices, the printer or network equipment |
| R4 | The printer must keep a fixed address |
| R5 | Network equipment must be managed only from trusted networks and only over an encrypted connection |
| R6 | Unused switch ports must not allow anyone to join the network |
| R7 | All wireless networks must be encrypted |
| R8 | Staff and guests must share one public address |

## 4. Topology

![Network topology](../screenshots/topology.png)

The office switch connects to the office router with a single trunk cable. The office router connects to a simulated ISP router, which connects to a web server representing the internet.

## 5. Device list

| Device | Role | Notes |
| --- | --- | --- |
| SOHO_R1 | Office router | VLAN gateways, DHCP, NAT, guest access list |
| SOHO_SW1 | Office switch | VLANs, trunk, port security |
| ISP_R1 | Provider router | Simulated internet provider |
| AP_Staff | Access point | Staff wireless network |
| AP_Guest | Access point | Guest wireless network |
| Staff_PC1, Staff_PC2 | Staff computers | DHCP |
| Printer1 | Shared printer | Fixed address |
| Staff_Phone | Staff wireless device | Joins the staff network |
| Guest_Tablet | Guest wireless device | Joins the guest network |
| WebServer | Simulated internet server | Fixed address |

## 5.1 Cable plan

| From | To | Cable |
| --- | --- | --- |
| SOHO_R1 Gi0/0 | SOHO_SW1 Gi0/1 | Straight through, trunk |
| SOHO_R1 Gi0/1 | ISP_R1 Gi0/0 | Crossover, WAN link |
| ISP_R1 Gi0/1 | WebServer Fa0 | Crossover |
| SOHO_SW1 Fa0/1 | Staff_PC1 | Straight through |
| SOHO_SW1 Fa0/2 | Staff_PC2 | Straight through |
| SOHO_SW1 Fa0/3 | Printer1 | Straight through |
| SOHO_SW1 Fa0/4 | AP_Staff | Straight through |
| SOHO_SW1 Fa0/5 | AP_Guest | Straight through |

## 6. VLAN plan

| VLAN | Name | Subnet | Gateway | Switch ports |
| --- | --- | --- | --- | --- |
| 10 | STAFF | 192.168.10.0/24 | 192.168.10.1 | Fa0/1, Fa0/2, Fa0/4 |
| 20 | GUEST | 192.168.20.0/24 | 192.168.20.1 | Fa0/5 |
| 30 | DEVICES | 192.168.30.0/24 | 192.168.30.1 | Fa0/3 |
| 99 | MANAGEMENT | 192.168.99.0/24 | 192.168.99.1 | Native VLAN on the trunk, switch address 192.168.99.2 |
| 999 | UNUSED | none | none | Fa0/6 to Fa0/24 and Gi0/2, shut down |

## 7. Addressing plan

| Item | Address |
| --- | --- |
| Router to ISP link, office side | 203.0.113.2/30 |
| Router to ISP link, ISP side | 203.0.113.1/30 |
| ISP router to web server network | 198.51.100.1/24 |
| Web server | 198.51.100.10 |
| Printer | 192.168.30.10 |
| Switch management | 192.168.99.2 |

DHCP pools run on the office router for VLANs 10, 20 and 30. The first 20 addresses of each pool network are excluded and kept for fixed devices, so leases start at .21.

## 8. Routing and NAT

1. The office router uses a default route pointing at the ISP router, 203.0.113.1.
2. NAT overload translates the Staff and Guest networks to the office router WAN address, 203.0.113.2.
3. The Devices network is not translated, because the printer has no need for internet access.

## 9. Security design

1. **Guest isolation:** an extended access list on the guest subinterface blocks traffic to the Staff, Devices and Management networks and permits everything else.
2. **Management access:** SSH version 2 only, with local accounts, allowed only from the Staff and Management networks.
3. **Credentials:** encrypted enable secret, hashed user secret and password encryption for any remaining passwords.
4. **Switch ports:** unused ports are shut down and moved to VLAN 999. Staff PC ports use port security with a maximum of two learned addresses.
5. **Wireless:** two separate networks using WPA2 with AES, one per access point.
6. **Trunk:** only VLANs 10, 20, 30 and 99 are allowed, with VLAN 99 as the native VLAN so untagged traffic stays off user networks.

## 10. Assumptions and limits

1. This is a simulation. No real internet service is involved.
2. The DHCP pools hand out the web server address as a DNS server, but no DNS service was configured, so tests use IP addresses.
3. Passwords are lab values and have no use outside the simulation.
4. Traffic filtering is limited to the guest network. Staff traffic is not restricted.
5. Packet Tracer does not model every real world behaviour, such as radio interference or firmware updates.

Back to the [README](../README.md).
