# Test Plan

Each test checks one requirement from the [Design Document](DesignDocument.md). Every test was run in Packet Tracer after the network was fully built and hardened. Evidence files are in the `screenshots` folder.

## Result key

1. **Pass:** the result matched the expected result.
2. **Fail:** the result did not match. A note below the table explains the fault and the fix.

## Connectivity tests

| ID | Requirement | Test | Expected result | Evidence | Result |
| --- | --- | --- | --- | --- | --- |
| T1 | R8 | From SOHO_R1, ping 203.0.113.1 and 198.51.100.10 | Replies from both. The first echo may time out | Not captured | |
| T2 | R1 | Run `show vlan brief` on SOHO_SW1 | STAFF holds Fa0/1, Fa0/2 and Fa0/4. GUEST holds Fa0/5. DEVICES holds Fa0/3 | `vlan_brief.png` | |
| T3 | R1 | Run `show interfaces trunk` on SOHO_SW1 | Gi0/1 trunking, VLANs 10, 20, 30 and 99 allowed, native VLAN 99 | `trunk.png` | |
| T4 | R1 | Run `show ip interface brief` on SOHO_R1 | Four Gi0/0 subinterfaces up with the gateway addresses. Gi0/1 shows 203.0.113.2 | `router_interfaces.png` | |
| T5 | R1 | Set Staff_PC1 to DHCP | Address begins with 192.168.10 with gateway 192.168.10.1 | `staff_pc_dhcp.png` | |
| T6 | R1, R4 | From Staff_PC1, ping the printer at 192.168.30.10 | Replies received | Not captured | |
| T7 | R1, R8 | From Staff_PC1, open the web server at 198.51.100.10 | Server page loads | `web_server_test.png` | |
| T8 | R8 | Run `show ip nat translations` on SOHO_R1 straight after a staff ping | Inside local addresses in 192.168.10 translated to 203.0.113.2 | `nat_translations.png` | |

## Wireless and guest tests

| ID | Requirement | Test | Expected result | Evidence | Result |
| --- | --- | --- | --- | --- | --- |
| T9 | R1, R7 | Connect Staff_Phone to SOHO_Staff | Address begins with 192.168.10 | `wifi_staff_connected.png` | |
| T10 | R2, R7 | Connect Guest_Tablet to SOHO_Guest | Address begins with 192.168.20 with gateway 192.168.20.1 | `wifi_guest_connected.png` | |
| T11 | R2 | From Guest_Tablet, ping 198.51.100.10 | Replies received | `guest_isolation.png` | |
| T12 | R3 | From Guest_Tablet, ping the printer at 192.168.30.10 | Request fails | `guest_isolation.png` | |
| T13 | R3 | From Guest_Tablet, ping a staff PC at 192.168.10.21 | Request fails | `guest_isolation.png` | |
| T14 | R3 | Run `show access-lists` on SOHO_R1 after T12 and T13 | GUEST_IN deny lines show match counts | `access_list_matches.png` | |

## Hardening tests

| ID | Requirement | Test | Expected result | Evidence | Result |
| --- | --- | --- | --- | --- | --- |
| T15 | R5 | Run `show ip ssh` on SOHO_R1 and SOHO_SW1 | SSH enabled, version 2.0 | `ssh_enabled.png` | |
| T16 | R5 | From Staff_PC1, run `ssh -l sohoadmin 192.168.10.1` | Login banner and router prompt after the password | `ssh_staff_login.png` | |
| T17 | R5 | From Guest_Tablet, run `ssh -l sohoadmin 192.168.20.1` | Connection refused, no password prompt | `ssh_guest_blocked.png` | |
| T18 | R6 | Run `show interfaces status` on SOHO_SW1 | Fa0/6 and up show disabled | `unused_ports_shutdown.png` | |
| T19 | R6 | Run `show port-security interface fastEthernet 0/1` on SOHO_SW1 | Port security enabled with a maximum of 2 | `port_security.png` | |
| T20 | R1 | From Staff_PC1, ping 198.51.100.10 after hardening | Replies received, network unaffected | Not captured | |

## Notes on failures and retests

Write one line here for each test that failed at first, what caused it, and how it was fixed. If every test passed first time, write that instead.

1. Several first pings timed out and then succeeded. This was the normal delay while devices learned neighbour addresses, so those tests were repeated.

## Summary

1. Total tests: 20
2. Passed: write the number here
3. Failed: write the number here

Back to the [README](../README.md).
