# Security Notes

This document explains the security controls in the SOHO network, which threat each one addresses, where the evidence is, and what the design does not protect against. Stating the limits is part of the work. A network is never fully secure, and a design that lists its gaps can be improved.

## 1. Trust zones

| Zone | Network | Trust level | Reason |
| --- | --- | --- | --- |
| Staff | VLAN 10 | High | Office computers and staff Wi-Fi |
| Devices | VLAN 30 | Medium | A printer that cannot defend itself |
| Management | VLAN 99 | Highest | Network equipment administration |
| Guest | VLAN 20 | Low | Unknown visitor devices |
| Internet | Simulated ISP | None | Outside the office |

## 2. Threats and controls

| Threat | Control | Where | Evidence |
| --- | --- | --- | --- |
| A guest device reaches staff computers | Extended access list GUEST_IN denies guest traffic to the Staff network | SOHO_R1, guest subinterface, inbound | T13, T14 |
| A guest device reaches the printer | The same access list denies the Devices network | SOHO_R1, guest subinterface, inbound | T12, T14 |
| A guest device reaches network equipment | The same access list denies the Management network, and SSH accepts logins only from Staff and Management addresses | SOHO_R1 and SOHO_SW1 | T14, T17 |
| Someone eavesdrops on remote administration | Telnet is disabled and only SSH version 2 is accepted | SOHO_R1 and SOHO_SW1 | T15, T16 |
| Someone guesses or reads a device password | Enable secret and user secret are stored as hashes, and password encryption covers any other stored passwords | SOHO_R1 and SOHO_SW1 | Exported configurations |
| An attacker plugs into a spare switch port | Unused ports are shut down and moved to an empty VLAN | SOHO_SW1 | T18 |
| Someone swaps a staff PC for another device | Port security with sticky learning and a maximum of two addresses | SOHO_SW1, ports Fa0/1 and Fa0/2 | T19 |
| Untagged traffic leaks onto user networks | The trunk carries only VLANs 10, 20, 30 and 99, with VLAN 99 as the native VLAN | SOHO_SW1 trunk | T3 |
| Someone joins the wireless networks without permission | WPA2 with AES on both access points | AP_Staff and AP_Guest | T9, T10 |
| An idle administration session is left open | Five minute idle timeout on remote login lines | SOHO_R1 and SOHO_SW1 | Exported configurations |
| Unauthorised login attempts are not warned off | Login banner stating that access is for authorised users only | SOHO_R1 and SOHO_SW1 | Exported configurations |
| The internal address plan is exposed to the internet | NAT overload hides every internal address behind 203.0.113.2 | SOHO_R1 | T8 |

## 3. Credential handling

1. All passwords in this project are simulation values with no use outside Packet Tracer.
2. The documents and build guide use placeholders, never real values.
3. The exported configuration files contain hashed secrets, not the original passwords. Each file was searched for the plain passwords before upload.
4. The Packet Tracer file stores the Wi-Fi passphrases. They were changed to lab only values before the file was shared.
5. In a real deployment, credentials would live in a password manager, be unique for every device, and be changed when staff leave.

## 4. Known limitations

1. **The access list is stateless.** It filters packet by packet and does not track connections. A staff computer that starts a conversation with a guest device would have the replies blocked. For this design that is acceptable, because staff and guests are not meant to talk to each other.
2. **Guests can still reach the router.** The final permit line allows guests to reach the guest gateway address and the router wide area address. Remote login is protected by the management access list, but the router still answers pings from guests.
3. **There is no filtering on the internet facing interface.** The office router has no inbound access list on its link to the ISP router. Only the lack of any published internal service and the remote login access list protect it.
4. **The management network is also the native VLAN.** Best practice uses a separate, unused native VLAN, so untagged frames never land in a network that matters. This design combined them to keep the plan small.
5. **Port security covers only the two staff PC ports.** The printer and access point ports have no limit on learned addresses.
6. **Wireless uses a shared passphrase.** Anyone who knows the staff passphrase can join the staff network. WPA2 PSK has no per user identity.
7. **Nothing is logged.** There is no syslog server, no time synchronisation and no record of who logged in or when.
8. **The console is only partly protected.** The console requires a login, but physical access to the equipment is not controlled.
9. **Spanning tree and Layer 2 protections are at defaults.** No DHCP snooping, ARP inspection or BPDU guard is configured, so a rogue DHCP server on a staff port would not be stopped.
10. **Discovery protocols are left on.** Cisco Discovery Protocol still shares device details with neighbours.
11. **No DNS service exists.** DHCP hands out an address for DNS that has no DNS service behind it, so all tests used IP addresses.
12. **This is a simulation.** Packet Tracer does not model real attacks, firmware weaknesses or radio level threats.

## 5. Improvements for a real deployment

1. Use WPA3 or WPA2 Enterprise with individual accounts for staff wireless.
2. Replace the stateless access list with a firewall that tracks connections, with a default deny rule on the internet facing side.
3. Move the native VLAN to a dedicated unused VLAN, separate from management.
4. Add DHCP snooping, dynamic ARP inspection and BPDU guard on access ports.
5. Send logs to a central syslog server and synchronise device clocks.
6. Apply port security to every active access port.
7. Turn off discovery protocols on ports that face users and the internet.
8. Keep device firmware up to date and schedule regular configuration backups.
9. Store credentials in a password manager, with unique values per device.
10. Review the configuration against a published hardening benchmark.

Back to the [README](../README.md).
