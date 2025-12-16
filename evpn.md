# EVPN Type-5 Systematic Test Plan

## Objective
Prove what is required for a VRF to advertise routes as EVPN Type-5 by testing each configuration element.

**Question:** Do routes need to be in the RIB of a VRF before that VRF can advertise them as Type-5?

---

## Test Environment
- Device: spine1 (aggr1)
- Source VRF: EXIT (has 11.11.11.11/32)
- Target VRF: OOB (will advertise to EVPN)
- Remote Peer: leaf-1 (10.1.1.3)

---

## TEST 0: Baseline

**Purpose:** Document current state before any changes

**Commands:**
```bash
nv show vrf EXIT router rib ipv4
nv show vrf OOB
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
sudo vtysh -c "show bgp l2vpn evpn summary" | grep 10.1.1.3
```

**Expected:**
- EXIT VRF has 11.11.11.11/32
- OOB VRF does not exist
- No Type-5 routes
- PfxSent to leaf-1: 0

---

## TEST 1: Create OOB VRF

**Purpose:** Create empty VRF container

**Commands:**
```bash
nv set vrf OOB table auto
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB
nv show vrf OOB router rib ipv4
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
```

**Expected:**
- VRF exists
- Only 0.0.0.0/0 route
- No Type-5 routes

**Proves:** Empty VRF does nothing for EVPN

---

## TEST 2: Add Loopback

**Purpose:** Add a local connected route

**Commands:**
```bash
nv set vrf OOB loopback ip address 10.3.1.1/32
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router rib ipv4
nv show vrf OOB router rib ipv4 | grep 10.3.1.1
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
```

**Expected:**
- 10.3.1.1/32 in RIB (connected)
- No Type-5 routes

**Proves:** Routes in RIB don't automatically advertise

---

## TEST 3: Enable BGP

**Purpose:** Enable BGP routing protocol in VRF

**Commands:**
```bash
nv set vrf OOB router bgp enable on
nv set vrf OOB router bgp autonomous-system 4260397300
nv set vrf OOB router bgp router-id 10.1.1.1
nv set vrf OOB router bgp address-family ipv4-unicast enable on
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router bgp
nv show vrf OOB router rib ipv4
sudo vtysh -c "show bgp vrf OOB ipv4 unicast"
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
```

**Expected:**
- BGP running in OOB VRF
- 10.3.1.1/32 still in RIB
- BGP table empty
- No Type-5 routes

**Proves:** BGP alone doesn't create Type-5 routes

---

## TEST 4: Import Routes from EXIT VRF

**Purpose:** CRITICAL TEST - Import 11.11.11.11 into OOB VRF RIB

**Commands:**
```bash
nv set vrf OOB router bgp address-family ipv4-unicast route-import from-vrf enable on
nv set vrf OOB router bgp address-family ipv4-unicast route-import from-vrf list EXIT
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router bgp address-family ipv4-unicast | grep route-import
nv show vrf OOB router rib ipv4
nv show vrf OOB router rib ipv4 | grep 11.11.11
sudo vtysh -c "show bgp vrf OOB ipv4 unicast"
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
nv show vrf OOB evpn
```

**Expected:**
- route-import enabled: YES
- 11.11.11.11/32 in OOB RIB: YES (Protocol: bgp)
- 10.2.1.x/32 in OOB RIB: YES
- Type-5 routes: NO
- EVPN enable: off

**Proves:** 🔑 **CRITICAL DISCOVERY**
- Routes ARE in VRF RIB
- But NOT advertised as Type-5
- Missing: route-export, L2VPN EVPN, L3 VNI

---

## TEST 5: Redistribute Connected

**Purpose:** Make connected routes available to BGP

**Commands:**
```bash
nv set vrf OOB router bgp address-family ipv4-unicast redistribute connected enable on
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router bgp address-family ipv4-unicast | grep redistribute
sudo vtysh -c "show bgp vrf OOB ipv4 unicast"
nv show vrf OOB router rib ipv4
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
```

**Expected:**
- redistribute enabled: YES
- 10.3.1.1/32 in BGP table: YES
- Type-5 routes: NO

**Proves:** Redistributing into BGP still doesn't trigger Type-5

---

## TEST 6: Enable route-export to-evpn

**Purpose:** Declare intent to export to EVPN

**Commands:**
```bash
nv set vrf OOB router bgp address-family ipv4-unicast route-export to-evpn enable on
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router bgp address-family ipv4-unicast | grep route-export
nv show vrf OOB evpn
nv show vrf OOB router rib ipv4 | grep 11.11.11
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
sudo vtysh -c "show bgp l2vpn evpn summary" | grep 10.1.1.3
```

**Expected:**
- route-export enabled: YES
- 11.11.11.11 in RIB: YES
- EVPN enable: off (no VNI)
- Type-5 routes: NO
- PfxSent to leaf-1: 0

**Proves:** route-export + routes in RIB still not enough

---

## TEST 7: Enable L2VPN EVPN

**Purpose:** Enable L2VPN EVPN address family

**Commands:**
```bash
nv set vrf OOB router bgp address-family l2vpn-evpn enable on
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB router bgp address-family l2vpn-evpn
nv show vrf OOB router rib ipv4
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
nv show vrf OOB evpn
```

**Expected:**
- L2VPN EVPN enabled: YES
- 11.11.11.11 in RIB: YES
- Type-5 routes: NO
- EVPN enable: still off

**Proves:** L2VPN EVPN capability doesn't trigger advertisement

---

## TEST 8: Enable EVPN (No VNI)

**Purpose:** Enable EVPN at VRF level WITHOUT VNI

**Commands:**
```bash
nv set vrf OOB evpn enable on
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB evpn
nv show vrf OOB evpn | grep vni
nv show vrf OOB router rib ipv4 | grep 11.11.11
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
sudo vtysh -c "show bgp l2vpn evpn summary" | grep 10.1.1.3
```

**Expected:**
- EVPN enable: on
- VNI: EMPTY ([vni] with no value)
- 11.11.11.11 in RIB: YES
- Type-5 routes: NO
- PfxSent to leaf-1: 0

**Proves:** 🔍 **THE SMOKING GUN**
- Everything configured EXCEPT VNI
- Routes in RIB: YES
- route-export: YES
- L2VPN EVPN: YES
- EVPN enabled: YES
- But NO Type-5 routes!
- L3 VNI is the missing piece

---

## TEST 9: Add L3 VNI

**Purpose:** THE CRITICAL TEST - Add L3 VNI and trigger Type-5

**Commands:**
```bash
nv set vrf OOB evpn vni 289001
nv set vrf OOB evpn vlan 3001
nv config apply -y
```

**Verify:**
```bash
nv show vrf OOB evpn
sudo vtysh -c "show evpn vni"
nv show vrf OOB router rib ipv4 | grep 11.11.11
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
sudo vtysh -c "show bgp l2vpn evpn route type prefix" | grep -A 5 11.11.11
sudo vtysh -c "show bgp l2vpn evpn summary" | grep 10.1.1.3
sudo vtysh -c "show bgp l2vpn evpn neighbors 10.1.1.3 advertised-routes"
sudo vtysh -c "show bgp l2vpn evpn summary"
```

**Expected:**
- VNI: 289001
- VLAN: 3001
- State: Up
- vxlan-interface: vxlan99
- VNI 289001 in "show evpn vni" (Type: L3, Tenant VRF: OOB)
- 11.11.11.11 still in RIB
- Type-5 routes: YES! (NOW APPEAR!)
  - [5]:[0]:[32]:[11.11.11.11]
  - RT:33012:289001
  - RD 10.1.1.1:4
- PfxSent to leaf-1: 5 (was 0!)
- 5 routes advertised to leaf-1

**Proves:** 🎉 **THE BREAKTHROUGH**
- ONLY change was adding L3 VNI
- Type-5 routes immediately appeared
- L3 VNI is THE TRIGGER

---

## Test Results Summary

| Test | Config Added | 11.11.11.11 in RIB? | Type-5 Advertised? | PfxSent |
|------|--------------|---------------------|-------------------|---------|
| 0 | Baseline | N/A | NO | 0 |
| 1 | VRF created | NO | NO | 0 |
| 2 | Loopback | NO | NO | 0 |
| 3 | BGP enabled | NO | NO | 0 |
| 4 | **route-import** | **YES** | **NO** | 0 |
| 5 | Redistribute | YES | NO | 0 |
| 6 | route-export | YES | NO | 0 |
| 7 | L2VPN EVPN | YES | NO | 0 |
| 8 | EVPN enable | YES | NO | 0 |
| 9 | **L3 VNI** | YES | **YES** | **5** |

---

## The Answer

**Question:** Do routes need to be in the RIB of a VRF before that VRF can advertise them as Type-5?

**Answer:** YES, but that's not sufficient. You need ALL THREE:

1. **Routes in VRF RIB** (proven in TEST 4)
2. **route-export to-evpn enabled** (proven in TEST 6)
3. **L3 VNI configured** (proven in TEST 9) ← THE TRIGGER

### The Proof

TEST 8 vs TEST 9:
- TEST 8: Routes in RIB ✓, export enabled ✓, NO VNI → NO Type-5
- TEST 9: Same + VNI added → Type-5 APPEAR

The ONLY difference was L3 VNI!

---

## Why L3 VNI Is Critical

L3 VNI provides:
1. **Route Target:** RT:ASN:VNI (e.g., RT:33012:289001)
2. **EVPN Control Plane:** Links VRF to EVPN infrastructure
3. **VXLAN Data Plane:** VNI for encapsulation
4. **Per-VNI Routing Table:** For EVPN route imports

Without VNI:
- No Route Target → No way to tag routes
- No EVPN integration → Routes stay local
- No VXLAN encap → Can't tunnel traffic

---

## Routes Advertised (After TEST 9)

5 routes now advertised as Type-5:

1. 10.2.1.1/32 (from EXIT via route-import)
2. 10.2.1.2/32 (from EXIT via route-import)
3. 10.2.1.3/32 (from EXIT via route-import)
4. 10.3.1.1/32 (local OOB loopback)
5. **11.11.11.11/32** (from EXIT via route-import)

All with RT:33012:289001

---

## Key Takeaways

### The Formula
```
Routes in RIB + route-export + L3 VNI = Type-5 Routes
```

Missing ANY ONE = No Type-5!

### Critical Tests
- **TEST 4:** Routes ARE in RIB but NOT advertised
- **TEST 8:** Everything except VNI → Still no Type-5
- **TEST 9:** Add VNI → Type-5 immediately appear

### Most Important Discovery
TEST 4 proved that "routes in RIB" alone is NOT sufficient. This is the most common misconception.

---

## Next Steps

Verify on leaf-1:

```bash
sudo vtysh -c "show bgp l2vpn evpn route type prefix"
nv show vrf OOB evpn  # Must have vni: 289001
nv show vrf OOB router rib ipv4
ip vrf exec OOB ping 11.11.11.11
```

If leaf-1 has OOB VRF with VNI 289001, routes will import and ping will work.

---

**Test Complete!**
