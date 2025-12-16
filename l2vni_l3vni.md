# Cumulus Linux VXLAN Infrastructure - vxlan48, vxlan99, and Bridge Behavior

## Overview

Cumulus Linux uses a **dual VXLAN device architecture** for separating Layer 2 and Layer 3 VNI traffic:
- **vxlan48** - Handles L2 VNIs (Layer 2 VXLAN)
- **vxlan99** - Handles L3 VNIs (Layer 3 VXLAN for symmetric routing)

Each VXLAN device belongs to its own bridge domain:
- **br_default** - Bridge for L2 VNIs (contains vxlan48)
- **br_l3vni** - Bridge for L3 VNIs (contains vxlan99)

---

## Device Architecture

### vxlan48 - Layer 2 VNI Device

**Purpose:**
- Single VXLAN device for all Layer 2 VNIs
- Handles MAC learning and Layer 2 forwarding
- Used for EVPN Type-2 routes (MAC/IP advertisement)

**Bridge:** br_default

**Characteristics:**
- Supports up to 4000 L2 VNIs
- Uses VLAN-to-VNI mapping
- Bridge learning typically disabled
- Created automatically when first L2 VNI configured

**Example Configuration:**
```bash
# Create L2 VNI
nv set bridge domain br_default vlan 200 vni 289200

# Result: Creates mapping in vxlan48
# VLAN 200 <-> VNI 289200
```

**Verification:**
```bash
ip link show vxlan48
# Output: vxlan48: ... master br_default ...

bridge vlan show dev vxlan48
# Shows VLAN-to-VNI mappings
```

---

### vxlan99 - Layer 3 VNI Device

**Purpose:**
- Single VXLAN device for all Layer 3 VNIs
- Handles inter-subnet routing (symmetric IRB)
- Used for EVPN Type-5 routes (IP prefix advertisement)

**Bridge:** br_l3vni

**Characteristics:**
- Created automatically when first L3 VNI configured
- Uses special VLAN range for L3 VNI mappings
- Bridge learning disabled
- Separate from L2 VNI infrastructure

**Example Configuration:**
```bash
# Enable global EVPN (triggers br_l3vni creation)
nv set evpn enable on

# Configure L3 VNI for a VRF
nv set vrf OOB evpn vni 289001
nv set vrf OOB evpn vlan 3001

# Result: Creates vxlan99 and br_l3vni
# VLAN 3001 <-> VNI 289001 in vxlan99
```

**Verification:**
```bash
ip link show vxlan99
# Output: vxlan99: ... master br_l3vni ...

bridge vlan show dev vxlan99
# Shows L3 VNI VLAN mappings

brctl show br_l3vni
# Shows bridge membership
```

---

## Bridge Domains

### br_default - Layer 2 Bridge

**Purpose:**
- Default VLAN-aware bridge
- Contains access ports, trunk ports, and vxlan48
- Handles Layer 2 switching and learning

**Members:**
- Physical switch ports (swp1, swp2, etc.)
- vxlan48 (L2 VNI device)
- Bond interfaces
- VLAN SVIs

**Characteristics:**
- VLAN-aware mode enabled
- Supports multiple VLANs
- MAC learning enabled on physical ports
- MAC learning disabled on VXLAN interface

**Example:**
```bash
nv show bridge domain br_default

brctl show br_default
# bridge name	bridge id		STP enabled	interfaces
# br_default	8000.xxxxxxxxxxxx	yes		swp1
#						swp2
#						vxlan48
```

---

### br_l3vni - Layer 3 Bridge

**Purpose:**
- Dedicated bridge for Layer 3 VNI infrastructure
- Contains only vxlan99
- Handles inter-subnet routing VLANs

**Members:**
- vxlan99 (L3 VNI device)
- Auto-created VLAN SVIs (e.g., vlan3001_l3)

**Characteristics:**
- VLAN-aware mode enabled
- Created automatically when `nv set evpn enable on`
- VLANs in this bridge are for L3 VNI use only
- Does not contain physical ports

**Example:**
```bash
brctl show br_l3vni
# bridge name	bridge id		STP enabled	interfaces
# br_l3vni	8000.xxxxxxxxxxxx	yes		vxlan99

bridge vlan show dev br_l3vni
# Shows L3 VNI VLANs (e.g., 3001, 3004)
```

---

## Observed Behavior: VLAN Auto-Assignment

### What You Observed

```bash
# Step 1: Set VNI without VLAN
nv set vrf OOB evpn vni 289001
nv config apply -y

bridge vlan
# port        vlan-id  
# vxlan99     2777      ← Auto-assigned VLAN!
# br_l3vni    1 PVID Egress Untagged
#             2777

# Step 2: Set explicit VLAN
nv set vrf OOB evpn vlan 3001
nv config apply -y

bridge vlan
# port        vlan-id  
# vxlan99     3001      ← Changed to explicit VLAN!
# br_l3vni    1 PVID Egress Untagged
#             3001
```

### Why This Happens

**Automatic VLAN Assignment:**
When you configure a L3 VNI without specifying a VLAN, NVUE automatically assigns a VLAN from the **reserved L3 VNI VLAN range**.

**Reserved VLAN Range:**
- Default range: **4000-4064** (for MLAG environments)
- Alternative range: **2700-2999** (for non-MLAG or different configs)
- The VLAN 2777 you saw is from this automatic allocation

**Check Reserved Range:**
```bash
nv show system global reserved vlan l3-vni-vlan
#       operational  applied
# -----  -----------  -------
# begin  4000         4000
# end    4064         4064
```

### Why Explicit VLAN Is Better

**Reasons to Specify VLAN:**
1. **Predictability** - You control the VLAN assignment
2. **Documentation** - Easier to track and troubleshoot
3. **Consistency** - Match VLAN across devices explicitly
4. **Avoid Conflicts** - Prevent overlap with other ranges

**Best Practice:**
```bash
# Always specify VLAN explicitly
nv set vrf <VRF> evpn vni <VNI>
nv set vrf <VRF> evpn vlan <VLAN>  # Explicit VLAN
```

---

## Complete VLAN-to-VNI Mapping Flow

### For L2 VNIs (vxlan48 in br_default)

```
Access Port (swp10)
  ↓ VLAN 200 tagged/untagged
br_default
  ↓ VLAN 200 → VNI 289200 mapping
vxlan48
  ↓ VXLAN encapsulation with VNI 289200
VXLAN Tunnel
```

**Configuration:**
```bash
# Create L2 VNI
nv set bridge domain br_default vlan 200 vni 289200

# Add access port
nv set interface swp10 bridge domain br_default access 200
```

**Result in vxlan48:**
```bash
bridge vlan show dev vxlan48
# vxlan48   200  ← VLAN 200 mapped to VNI 289200
```

---

### For L3 VNIs (vxlan99 in br_l3vni)

```
VRF OOB
  ↓ L3 VNI 289001
VLAN 3001 (L3 VNI VLAN)
  ↓ vlan3001_l3 SVI in VRF OOB
br_l3vni
  ↓ VLAN 3001 → VNI 289001 mapping
vxlan99
  ↓ VXLAN encapsulation with VNI 289001
VXLAN Tunnel (for Type-5 routes)
```

**Configuration:**
```bash
# Configure L3 VNI
nv set vrf OOB evpn enable on
nv set vrf OOB evpn vni 289001
nv set vrf OOB evpn vlan 3001
```

**Result in vxlan99:**
```bash
bridge vlan show dev vxlan99
# vxlan99   3001  ← VLAN 3001 mapped to VNI 289001

bridge vlan show dev br_l3vni
# br_l3vni  1 PVID Egress Untagged
#           3001  ← L3 VNI VLAN in bridge
```

**Auto-Created SVI:**
```bash
ip link show vlan3001_l3
# vlan3001_l3@br_l3vni: ... master OOB ...
# ↑ Automatically created SVI in VRF OOB
```

---

## Why Separate Bridges and VXLAN Devices?

### Design Rationale

**Separation of Concerns:**
1. **L2 VNIs (vxlan48/br_default):**
   - Handle host connectivity
   - MAC learning and forwarding
   - Bridging within subnets
   
2. **L3 VNIs (vxlan99/br_l3vni):**
   - Handle routing between subnets
   - No host MACs (only router MACs)
   - Type-5 prefix routes

**Operational Benefits:**
- **Isolation:** L2 and L3 traffic separated
- **Scalability:** Different VLAN ranges, no conflicts
- **Troubleshooting:** Clear separation of Layer 2 vs Layer 3 issues
- **Performance:** Optimized for different traffic patterns

**MLAG Consideration:**
In MLAG environments, you typically use a single bridge (br_default) for both L2 and L3 VNIs because the MLAG peerlink must be in the same bridge. However, in non-MLAG collapsed spine/leaf or pure leaf deployments, separate bridges are used.

---

## Verification Commands

### Check VXLAN Devices

```bash
# Show all VXLAN interfaces
ip link show type vxlan

# Expected output:
# vxlan48: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 ... master br_default
# vxlan99: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 ... master br_l3vni
```

### Check Bridges

```bash
# Show bridge membership
brctl show

# Expected:
# br_default   ... swp1 swp2 vxlan48
# br_l3vni     ... vxlan99
```

### Check VLAN Mappings

```bash
# L2 VNI mappings
bridge vlan show dev vxlan48
# vxlan48   200  (VNI 289200)

# L3 VNI mappings
bridge vlan show dev vxlan99
# vxlan99   3001  (VNI 289001)
#           3004  (VNI 289004)

# Bridge VLAN membership
bridge vlan show dev br_l3vni
# br_l3vni  1 PVID Egress Untagged
#           3001
#           3004
```

### Check VNI Status

```bash
# Show all VNIs
sudo vtysh -c "show evpn vni"

# Expected:
# VNI    Type  VxLAN IF  Tenant VRF
# 289200 L2    vxlan48   OOB
# 289001 L3    vxlan99   OOB
# 289004 L3    vxlan99   EXIT
```

### Check Auto-Created SVIs

```bash
# List L3 VNI SVIs
ip link show | grep _l3

# vlan3001_l3@br_l3vni: ...
# vlan3004_l3@br_l3vni: ...

# Check SVI VRF membership
ip link show vlan3001_l3

# vlan3001_l3@br_l3vni: ... master OOB ...
```

---

## Configuration Examples

### Complete L2 VNI Setup (vxlan48)

```bash
# Enable VXLAN
nv set nve vxlan enable on
nv set nve vxlan source address 10.1.1.1

# Create L2 VNI
nv set bridge domain br_default vlan 200 vni 289200

# Add access port
nv set interface swp10 bridge domain br_default access 200

# Apply
nv config apply -y

# Verify
bridge vlan show dev vxlan48  # Should show VLAN 200
```

### Complete L3 VNI Setup (vxlan99)

```bash
# Enable global EVPN (creates br_l3vni and vxlan99)
nv set evpn enable on

# Configure VRF with L3 VNI
nv set vrf OOB evpn enable on
nv set vrf OOB evpn vni 289001
nv set vrf OOB evpn vlan 3001  # Explicit VLAN

# Configure BGP for EVPN
nv set vrf OOB router bgp enable on
nv set vrf OOB router bgp autonomous-system 4260397300
nv set vrf OOB router bgp router-id 10.1.1.1
nv set vrf OOB router bgp address-family l2vpn-evpn enable on
nv set vrf OOB router bgp address-family ipv4-unicast route-export to-evpn enable on

# Apply
nv config apply -y

# Verify
bridge vlan show dev vxlan99  # Should show VLAN 3001
sudo vtysh -c "show evpn vni"  # Should show VNI 289001 Type L3
```

---

## Summary: Key Takeaways

### VXLAN Device Separation

| Device | Purpose | Bridge | VNI Type | Use Case |
|--------|---------|--------|----------|----------|
| vxlan48 | L2 VNIs | br_default | Layer 2 | Host connectivity, Type-2 routes |
| vxlan99 | L3 VNIs | br_l3vni | Layer 3 | Inter-subnet routing, Type-5 routes |

### VLAN Behavior

1. **Automatic Assignment:**
   - If you don't specify VLAN, NVUE assigns from reserved range
   - Example: VNI 289001 got VLAN 2777 automatically

2. **Explicit Assignment (Recommended):**
   - Always specify VLAN with `nv set vrf <VRF> evpn vlan <VLAN>`
   - Provides predictability and consistency
   - Example: VNI 289001 → VLAN 3001 (explicit)

3. **VLAN Ranges:**
   - L2 VNI VLANs: User-defined (e.g., 200, 300, 400)
   - L3 VNI VLANs: Reserved range (2700-2999 or 4000-4064)

### Bridge Isolation

- **br_default:** Physical ports + vxlan48 (L2 traffic)
- **br_l3vni:** Only vxlan99 (L3 routing)
- Separation provides clear traffic segregation

### Auto-Created Resources

When you configure L3 VNI:
- vxlan99 device created (if first L3 VNI)
- br_l3vni bridge created (if first L3 VNI)
- vlan<VLAN>_l3 SVI created automatically
- VLAN added to br_l3vni
- VLAN-to-VNI mapping added to vxlan99

---

## Troubleshooting

### vxlan48 Not Created

**Issue:** vxlan48 doesn't exist

**Check:**
```bash
nv show bridge domain br_default vlan
```

**Solution:**
```bash
# Create at least one L2 VNI
nv set bridge domain br_default vlan 200 vni 289200
nv config apply
```

### vxlan99 Not Created

**Issue:** vxlan99 doesn't exist

**Check:**
```bash
nv show evpn
```

**Solution:**
```bash
# Enable global EVPN
nv set evpn enable on
nv config apply
```

### Wrong VLAN Assignment

**Issue:** L3 VNI got unexpected VLAN (like 2777)

**Check:**
```bash
bridge vlan show dev vxlan99
```

**Solution:**
```bash
# Set explicit VLAN
nv set vrf <VRF> evpn vlan <desired-vlan>
nv config apply
```

### Bridge Membership Issues

**Issue:** Interface in wrong bridge

**Check:**
```bash
brctl show
ip link show <interface> | grep master
```

**Solution:**
```bash
# For L2 VNI - should be in br_default
nv set bridge domain br_default vlan <vlan> vni <vni>

# For L3 VNI - automatically goes to br_l3vni
nv set vrf <VRF> evpn vni <vni>
```

---

## Related Documentation

- Cumulus Linux VXLAN Devices: https://docs.nvidia.com/networking-ethernet-software/cumulus-linux/Network-Virtualization/VXLAN-Devices/
- Inter-subnet Routing: https://docs.nvidia.com/networking-ethernet-software/cumulus-linux/Network-Virtualization/Ethernet-Virtual-Private-Network-EVPN/Inter-subnet-Routing/
- VLAN-aware Bridge Mode: https://docs.nvidia.com/networking-ethernet-software/cumulus-linux/Layer-2/Ethernet-Bridging-VLANs/VLAN-aware-Bridge-Mode/
