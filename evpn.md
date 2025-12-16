# EVPN Type-5 Route Troubleshooting Summary

## Problem Statement
After configuring VRF OOB with EVPN settings, no EVPN Type-5 prefix routes were being advertised, despite having a working VRF EXIT configuration.

## Initial State
- VRF EXIT was working correctly with EVPN routes visible
- Attempted to configure VRF OOB with similar settings

## Configuration Steps Taken

### 1. Initial VRF OOB Configuration
```bash
nv set vrf OOB evpn enable on
nv set vrf OOB loopback ip address 10.3.1.1/32
nv set vrf OOB router bgp address-family ipv4-unicast enable on
nv set vrf OOB router bgp address-family ipv4-unicast redistribute connected enable on
nv set vrf OOB router bgp address-family ipv4-unicast route-export to-evpn enable on
nv set vrf OOB router bgp address-family ipv4-unicast route-import from-vrf enable on
nv set vrf OOB router bgp address-family ipv4-unicast route-import from-vrf list EXIT
nv set vrf OOB router bgp address-family l2vpn-evpn enable on
nv set vrf OOB router bgp autonomous-system 4260397300
nv set vrf OOB router bgp enable on
nv set vrf OOB router bgp router-id 10.1.1.1
nv config apply -y
```

**Result:** BGP routes appeared in VRF OOB, but EVPN Type-5 routes did NOT appear
- Command `sudo vtysh -c "show bgp l2vpn evpn route type prefix"` returned: "No EVPN prefixes (of requested type) exist"

### 2. Added EVPN VNI and VLAN
```bash
nv set vrf OOB evpn vni 289001
nv set vrf OOB evpn vlan 3001
nv config apply -y
```

**Result:** Still no EVPN Type-5 routes appearing

### 3. THE CRITICAL FIX: Enable Global EVPN

```bash
nv set evpn enable on
nv config apply -y
```

**Result:** SUCCESS! EVPN Type-5 routes immediately appeared with 10 prefixes advertised

## Root Cause Analysis

### The Missing Piece
**Global EVPN was disabled.** Even though:
- VRF EXIT had EVPN enabled and configured
- VRF OOB had EVPN enabled with proper VNI/VLAN
- All BGP address-family settings were correct

The global EVPN feature switch was OFF, preventing any EVPN route advertisements.

### Verification
This was confirmed by toggling the global EVPN setting:

```bash
# Disabled global EVPN
nv set evpn enable off
nv config apply -y
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
# Result: "No EVPN prefixes (of requested type) exist"

# Re-enabled global EVPN
nv set evpn enable on
nv config apply -y
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
# Result: 10 prefixes displayed with both Route Distinguishers (10.1.1.1:2 and 10.1.1.1:4)
```

## Final Working Configuration

The complete working configuration requires:

### Per-VRF Settings (VRF EXIT and VRF OOB)
- EVPN enabled: `nv set vrf <VRF> evpn enable on`
- VNI assignment: `nv set vrf <VRF> evpn vni <vni-id>`
- VLAN assignment: `nv set vrf <VRF> evpn vlan <vlan-id>`
- Loopback address for VRF
- BGP configuration with:
  - IPv4 unicast address-family
  - L2VPN EVPN address-family
  - Route export to EVPN
  - Route import from other VRFs (if inter-VRF routing needed)
  - Connected route redistribution

### Global Setting (THE KEY!)
- **Global EVPN must be enabled:** `nv set evpn enable on`

## EVPN Type-5 Routes Advertised

After enabling global EVPN, the following Type-5 routes were successfully advertised:

### Route Distinguisher: 10.1.1.1:2 (VRF EXIT - VNI 289004)
- 10.2.1.1/32
- 10.2.1.2/32
- 10.2.1.3/32
- 10.3.1.1/32
- 11.11.11.11/32

### Route Distinguisher: 10.1.1.1:4 (VRF OOB - VNI 289001)
- 10.2.1.1/32 (imported from EXIT)
- 10.2.1.2/32 (imported from EXIT)
- 10.2.1.3/32 (imported from EXIT)
- 10.3.1.1/32 (local)
- 11.11.11.11/32 (imported from EXIT)

## Lessons Learned

1. **Global EVPN enable is required** - Per-VRF EVPN settings are not sufficient
2. **Hierarchical configuration** - Check both global and per-VRF settings when troubleshooting
3. **The command `nv show evpn` can verify global EVPN status** before diving into per-VRF troubleshooting
4. **Route targets** (RT:33012:289004 and RT:33012:289001) are automatically generated based on VNI configuration

## Summary

**The single critical change that made everything work was:**
```bash
nv set evpn enable on
```

Without this global setting, no EVPN Type-5 routes are advertised, regardless of how correctly the per-VRF EVPN and BGP settings are configured.

