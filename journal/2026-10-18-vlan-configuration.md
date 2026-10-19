# 2026-10-18: Configuring VLANs

Working through the [VLAN plan](../docs/networking/vlan-plan.md). This entry is in progress.

## Approach

The VLANs are created on the ER605 first, not on the switch alone. The switch can separate Ethernet traffic, but the ER605 still has to route that traffic and provide DHCP for each network. (A different architecture could change this, but that would be a deliberate choice.)

## Steps taken

### Finding the network list on the ER605 (plan step 3)

1. Went back to the ER605 web interface.
2. Opened **Network → LAN**.
3. Found the **Network List** (the list of LAN networks; the exact name may differ by firmware).
4. Located the **Add** button for creating a new network.

### Step 1: Create the first VLAN network

In **Network → LAN → Network List**:

1. Click **Add**.
2. Look for fields such as Network Name, VLAN ID, IP Address, Subnet Mask, and DHCP Server.
3. Enter these settings:

| Setting | Value |
| --- | --- |
| Network Name | Trusted |
| VLAN ID | 10 |
| IP Address | 192.168.10.1 |
| Subnet Mask | 255.255.255.0 |
| DHCP Server | Enabled |
| DHCP Range | 192.168.10.100 to 192.168.10.199 |

**Leave the existing LAN network unchanged.** This adds a separate network; it does not replace the one currently giving the Mac internet access.

### Step 2: Create the Lab and IoT networks

Same place: **Network → LAN → Network List → Add**. Keep **Network Isolation disabled** on both for now; separation will come from firewall rules later.

**VLAN 20: Lab.** Servers, mini PCs, virtual machines, and cybersecurity practice environments.

| Setting | Value |
| --- | --- |
| Network Name | Lab |
| VLAN ID | 20 |
| IP Address | 192.168.20.1 |
| Subnet Mask | 255.255.255.0 |
| DHCP Server | Enabled |
| DHCP Start IP | 192.168.20.100 |
| DHCP End IP | 192.168.20.199 |
| Network Isolation | Disabled |

**VLAN 30: IoT.** Smart TV and other smart-home devices.

| Setting | Value |
| --- | --- |
| Network Name | IoT |
| VLAN ID | 30 |
| IP Address | 192.168.30.1 |
| Subnet Mask | 255.255.255.0 |
| DHCP Server | Enabled |
| DHCP Start IP | 192.168.30.100 |
| DHCP End IP | 192.168.30.199 |
| Network Isolation | Disabled |

### Step 3: Create the Apps network

Same place: **Network → LAN → Network List → Add**.

**VLAN 40: Apps.** Applications and self-hosted services, such as a media server, dashboard, password manager, or reverse proxy.

| Setting | Value |
| --- | --- |
| Network Name | Apps |
| VLAN ID | 40 |
| IP Address / Gateway | 192.168.40.1 |
| Subnet Mask | 255.255.255.0 |
| DHCP Server | Enabled |
| DHCP Start IP | 192.168.40.100 |
| DHCP End IP | 192.168.40.199 |
| Network Isolation | Disabled |

## Mistake: every switch port set to tagged

### What happened

While setting up 802.1Q VLANs on the TL-SG108E, I set **all ports as tagged** for the VLAN. Right after saving, both the ER605 and the Mac lost their connection through the switch, and I could no longer reach the switch's admin page. I had to **factory reset the switch** to recover.

### Why it broke

A tagged port sends frames with a VLAN tag added, and expects tagged frames back. Only devices that understand VLAN tags (like the ER605's port to the switch, or a VLAN-aware AP or hypervisor) can use a tagged port.

- The **Mac** sends and expects normal, untagged frames. It can't use a tagged port.
- The **ER605's existing LAN** (the default network) is untagged on port 2, so it also stopped matching what the switch was sending.
- The **switch's admin page** lives on VLAN 1, so once nothing reached it untagged, there was no way back in.

### The rule

| Port type | Example | Setting |
| --- | --- | --- |
| Trunk (to a VLAN-aware device) | Switch port 1 to ER605 port 2 | Untagged on VLAN 1; **tagged** on 10, 20, 30, 40 |
| Access (to a normal device) | Mac, server, smart TV | **Untagged** on its one VLAN, with the PVID set to match |
| Admin | Switch port 8 | Untagged on VLAN 1 |

Tag only on links to devices that understand tags. Everything else is untagged.

### Lessons

- **Back up the switch config before changing VLANs.** That step was still open when this happened.
- **Do the setup from the admin port (port 8)** and never change that port's VLAN settings during setup.
- **Change one port at a time**, starting with ports nothing important is plugged into, and check the connection after each change.
- **A factory reset is the recovery path** if locked out. On this switch, it's the reset pinhole on the device. The ER605 was not reset, so its configuration was unaffected.
- **After a reset, check the switch's IP address.** Its factory default may be `192.168.0.1`, the same as the ER605's gateway. If the switch doesn't pick up a DHCP address, give it a free address in `192.168.0.x` before going further.

## Recovery confirmed after factory reset

Post-reset connectivity check, with the Mac on the admin port as planned:

| Link | Result |
| --- | --- |
| MacBook → TL-SG108E, port 8 | IP address `192.168.0.101` |
| TL-SG108E → ER605, port 1 | Internet access confirmed |
| Switch management | Web interface accessible |

The switch picked up `192.168.0.101` from the ER605's DHCP server, so there was no IP collision with the ER605's own `192.168.0.1` this time. Back to a known-good state, ready to retry the VLAN setup more carefully.

## VLAN 10 configured and tested successfully

Applied the lesson from the earlier mistake: tagged only the trunk port, left every device-facing port untagged.

### Switch VLAN settings

| VLAN | Port 1 (to ER605) | Port 2 (Mac) | Port 8 (admin) |
| --- | --- | --- | --- |
| 1 (default) | Untagged | — | Untagged |
| 10 (Trusted) | Tagged | Untagged | — |

This is the opposite of the earlier mistake: last time VLAN 1 was tagged everywhere, including on the ports the Mac and the switch admin page depend on being untagged. This time VLAN 1 stayed untagged on ports 1 and 8, and only VLAN 10 was tagged, and only on port 1.

### Test

Moved the Mac to port 2. It came up on VLAN 10 with internet access. **Success.**

## All VLANs configured on the switch and tested

Added the remaining VLANs to the TL-SG108E and manually tested each by plugging a device into its port. All working. Port 8 is reserved for management (admin access, untagged on VLAN 1, not used for test traffic).

| Port | VLAN | Result |
| --- | --- | --- |
| 2 | 10 (Trusted) | Working (tested earlier with the Mac) |
| 3 | 20 (Lab) | Working |
| 4 | 40 (Apps) | Working |
| 5 | 30 (IoT) | Working |
| 8 | 1 (Management) | Reserved, not used for VLAN test traffic |

Port 5 — originally reserved in the plan for a future Wi-Fi AP — was used to test VLAN 30 (IoT). Final port assignments, including where the AP will land, are still open.

Ports 1 (trunk to ER605) and 8 (management) are the only two with a fixed role; 2–5 are now each dedicated to one VLAN for testing, with 6–7 still spare.

## Step 4: Firewall rules between VLANs (in progress)

VLANs separate the networks; they don't restrict traffic between them on their own. The ER605 routes between VLANs by default, so deliberate firewall rules are what actually enforce isolation between Trusted, Lab, Apps, and IoT.

Full rule matrix, including which "No" rows are assumed defaults vs. confirmed, is in the [VLAN plan](../docs/networking/vlan-plan.md#inter-vlan-firewall-rules). Short version: Trusted can reach everything; Lab, IoT, and Apps are denied from reaching Trusted, Management, and each other by default.

### To do on the ER605

1. Find the firewall / ACL rule screen (exact menu name TBD — report back what it's called and what fields it has).
2. Confirm whether unmatched traffic is allowed or denied by default, and whether the firewall is stateful (tracks return traffic for allowed connections automatically).
3. Add deny rules for each "No" pair in the plan's matrix, most specific first.
4. Test from each VLAN: confirm Trusted can still reach Lab/IoT/Apps/Management, and confirm Lab/IoT/Apps cannot reach Trusted/Management or each other.
5. Record the actual rules created (name, source, destination, action) here once done.

## Checklist

- [ ] Back up the ER605 and TL-SG108E configs (plan step 1)
- [x] Mac on switch port 8, the VLAN 1 admin port (plan step 2)
- [ ] Create VLAN 10 (Trusted) on the ER605
- [ ] Create VLAN 20 (Lab) on the ER605
- [ ] Create VLAN 30 (IoT) on the ER605
- [ ] Create VLAN 40 (Apps) on the ER605
- [ ] Tag VLANs 10, 20, 30 and 40 on ER605 port 2
- [x] Confirm the switch is reachable again after the factory reset, and note its IP address (`192.168.0.101`)
- [ ] Back up the switch config before retrying
- [x] Configure 802.1Q VLANs and PVIDs on the TL-SG108E (second attempt, one port at a time)
- [x] Test the Mac on switch port 2 (VLAN 10) — success, internet access confirmed
- [x] Configure VLAN 20 (Lab) on the switch (port 3) and test — working
- [x] Configure VLAN 40 (Apps) on the switch (port 4) and test — working
- [x] Configure VLAN 30 (IoT) on the switch (port 5) and test — working
- [ ] Decide final port for the Wi-Fi AP (was reserved as port 5; now port 5 is testing VLAN 30)
- [ ] Find and document the ER605 firewall/ACL rule screen
- [ ] Confirm default allow/deny behavior and whether the firewall is stateful
- [ ] Add inter-VLAN firewall rules (see matrix in the plan)
- [ ] Test that Trusted can still reach Lab/IoT/Apps/Management
- [ ] Test that Lab, IoT, and Apps cannot reach Trusted, Management, or each other

## Notes

- Firmware version and exact menu names: _fill in_
