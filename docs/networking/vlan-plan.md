# VLAN plan

**Status:** All four VLANs created on the ER605 and configured on the switch; each one tested working on a dedicated port (2026-10-18). Final device-to-port assignments and firewall rules are still open. See the [configuration journal](../../journal/2026-10-18-vlan-configuration.md).

Builds on the [initial design](../architecture/initial-design.md). The physical network is already working end to end ([journal](../../journal/2026-10-18-physical-network-online.md)).

## What a VLAN needs

A VLAN is more than an ID. Each one needs three things to work as intended:

1. **Routing and DHCP on the ER605.** The router provides the VLAN's gateway and hands out addresses in its subnet.
2. **Port placement on the TL-SG108E.** The switch puts each port into the correct VLAN, and carries tagged traffic for every VLAN on the link to the router.
3. **Firewall rules on the ER605.** Devices in different VLANs are isolated only when the router's firewall rules enforce that separation. Without rules, the ER605 routes freely between VLANs.

## Current setup

| Item | Value |
| --- | --- |
| ER605 mode | Standalone (configured in its own web UI, no Omada controller) |
| ER605 port to switch | Port 2 (LAN) |
| Switch port to ER605 | Port 1 |
| Wi-Fi access point | Not bought yet |

## VLANs

The VLAN ID matches the third octet of the subnet, so an address shows its VLAN at a glance.

| VLAN ID | Name | Subnet | Gateway (ER605) | DHCP range | Purpose | Status |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Management | 192.168.0.0/24 | 192.168.0.1 | .100 to .199 | Router and switch admin (ER605 default LAN) | Exists |
| 10 | Trusted | 192.168.10.0/24 | 192.168.10.1 | .100 to .199 | PCs and laptops | Configure now |
| 20 | Lab | 192.168.20.0/24 | 192.168.20.1 | .100 to .199 | Servers, mini PCs, VMs, and cybersecurity practice environments | Configure now |
| 30 | IoT | 192.168.30.0/24 | 192.168.30.1 | .100 to .199 | Smart TV and other smart-home devices | Configure now |
| 40 | Apps | 192.168.40.0/24 | 192.168.40.1 | .100 to .199 | Self-hosted services: media server, dashboard, password manager, reverse proxy | Configure now |

Network Isolation is left disabled on the ER605 for all of these for now. Separation between VLANs will come from firewall rules (step 10).

Addresses below `.100` in each subnet are left for static assignments and DHCP reservations.

## Switch port map (TL-SG108E)

| Port | Connects to | Mode | Untagged VLAN (PVID) | Tagged VLANs | Status |
| --- | --- | --- | --- | --- | --- |
| 1 | ER605 port 2 | Trunk | 1 | 10, 20, 30, 40 | Configured and tested 2026-10-18 |
| 2 | Trusted test device | Access | 10 | | Configured and tested 2026-10-18 |
| 3 | Lab test device | Access | 20 | | Configured and tested 2026-10-18 |
| 4 | Apps test device | Access | 40 | | Configured and tested 2026-10-18 |
| 5 | IoT test device | Access | 30 | | Configured and tested 2026-10-18 |
| 6 | Spare | Access | _TBD_ | | Not yet assigned |
| 7 | Spare | Access | _TBD_ | | Not yet assigned |
| 8 | Admin / management port | Access | 1 | | Reserved, in use for management |

**Port assignments above reflect testing, not final device placement.** Ports 2–5 were each used to confirm one VLAN works, not to permanently assign a device. The homelab server, Mac, and future Wi-Fi AP still need to be assigned to final ports — see the open question below.

Port 8 stays on VLAN 1 as a fallback way into the router and switch admin pages.

## Configuration order

The order is designed to avoid locking yourself out. Menu names vary by firmware and hardware version.

1. **Back up both configs.** Export the ER605 config and the TL-SG108E config before changing anything.
2. **Move the Mac to switch port 8.** Do the setup from the admin port so the Mac stays on VLAN 1 throughout.
3. **ER605: create the networks.** Add VLANs 10, 20, 30 and 40, each with its gateway IP and DHCP server. Leave the existing LAN unchanged and Network Isolation disabled.
4. **ER605: tag the VLANs on port 2.** VLAN 1 stays untagged; 10, 20, 30 and 40 are tagged.
5. **TL-SG108E: enable 802.1Q VLAN.** Create VLANs 10, 20, 30 and 40.
6. **TL-SG108E: set port membership.** Port 1: untagged 1, tagged 10, 20, 30 and 40. Access ports: untagged in their one VLAN.
   **Do not set every port to tagged.** Only port 1 (to the ER605) carries tags. Tagging access ports cuts off normal devices and the switch admin page; this happened once and needed a factory reset ([journal](../../journal/2026-10-18-vlan-configuration.md#mistake-every-switch-port-set-to-tagged)). Change one port at a time and leave port 8 alone.
7. **TL-SG108E: set the PVID** on each access port to match its VLAN.
8. **Test.** Move the Mac to port 2. It should get a `192.168.10.x` address and reach the internet.
9. **Repeat the test for VLAN 20** once the homelab server is connected to port 3.
10. **Add firewall rules between VLANs** (below). Until then, the VLANs are separate subnets but not isolated.
11. **Update this doc** to "Configured" and record the date in the journal.

## Inter-VLAN firewall rules

VLANs only separate broadcast domains and subnets. The ER605 still routes between them by default, so nothing is actually isolated until firewall rules say so. This is the step that enforces the intended boundaries.

**Design:** Trusted is the admin/control zone and can reach everything it needs to use or manage. Lab, IoT, and Apps are isolated from Trusted, from Management, and from each other by default — deny first, then add narrow allows later only for traffic that's actually needed (e.g., a specific port from IoT to a media server), rather than opening a whole zone.

| From | To | Allow? |
| --- | --- | --- |
| Trusted | Management | Yes |
| Trusted | Lab | Yes |
| Trusted | IoT | Yes |
| Trusted | Apps | Yes |
| Lab | Trusted | **No** |
| Lab | Management | **No** |
| Lab | IoT | **No** *(assumed — keeps lab/practice traffic away from real IoT devices)* |
| Lab | Apps | **No** *(assumed — tighten later if lab workloads need to reach self-hosted services)* |
| IoT | Trusted | **No** |
| IoT | Lab | **No** |
| IoT | Management | **No** |
| IoT | Apps | **No** *(assumed — e.g. smart TV to a media server; revisit as a narrow, specific rule if casting/streaming needs it, not a blanket IoT→Apps allow)* |
| Apps | Trusted | **No** |
| Apps | Lab | **No** |
| Apps | IoT | **No** *(assumed — revisit if a self-hosted app needs to control IoT devices, e.g. a home automation hub)* |
| Apps | Management | **No** |
| All | Internet | Yes |

Rows marked *assumed* are my default, least-privilege guess, not something you've confirmed — correct any of them if you intend different access.

**Established connections:** allowing Trusted → Lab while denying Lab → Trusted normally still needs *return* traffic from Lab permitted for connections Trusted opened. Check whether the ER605's ACL is stateful (tracks connections automatically) or whether it needs that handled explicitly — note what you see when you get to the rule screen.

**Rule order:** on most SOHO/SMB firewalls, including the ER605, ACL rules are evaluated top-to-bottom, and unmatched traffic is allowed by default. That means only the **No** rows above need an explicit **Deny** rule; the **Yes** rows need nothing added, since they're already permitted unless something denies them. Put deny rules at the top, most specific first, so they aren't skipped by a broader allow. Confirm this default-allow behavior when you're in the UI, since it changes the approach if the ER605 instead defaults to deny.

## When the AP is bought

- Look for an AP that supports multiple SSIDs mapped to VLANs (802.1Q tagging).
- Map SSIDs to VLANs (for example, a main SSID on VLAN 10 and an IoT SSID on VLAN 30), and make switch port 5 a trunk carrying those VLANs.
- Move the wireless devices to the AP, then turn off the BGW320's Wi-Fi.

## Open question: final port assignments

Ports 2–5 were used to test each VLAN individually, not to assign final devices. Before relying on this permanently, decide:

- Which port the Mac (Trusted) stays on long-term
- Which port the homelab server (Lab) uses
- Which port the smart TV / IoT devices use
- Which port hosts Apps services
- Whether to still reserve a port for the future Wi-Fi AP (originally planned as port 5, now used for the IoT test)

## Known gaps

- **BGW320 Wi-Fi is still on.** It will be turned off once the AP is in place for wireless devices. Until then, clients on the BGW320's Wi-Fi bypass the ER605, and none of these VLANs or firewall rules apply to them.
