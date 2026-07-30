---
Title: "Exercise 5: IPv6 Routing"
Subtitle: "Extending an L2 fabric into an IP fabric"
---

# Exercise 5: IPv6 Routing

**Objective:** enable hosts in different subnets to have **IPv6 routing** across a
leaf and spine fabric, with Equal-Cost Multi-Path (ECMP) load balancing
across the spine switches.

Scope: modifications to the P4 pipeline and to the ONOS control application.

---

## Starting Point: A Pure L2 Bridge

Prior to this exercise, each switch forwards traffic solely on the basis of
the Ethernet MAC address:

- `l2_exact_table`: exact match on `eth.dst`, producing an output port.
- `l2_ternary_table`: match on `eth.dst`, producing a flood for broadcast
  and NDP traffic.

The switch does not inspect the IPv6 header. It has no concept of a route, a
next hop, or a gateway. This is sufficient within a single subnet but fails
between subnets.

---

## Why Same-Subnet Communication Works Today

Two hosts on the same `/64` prefix reach each other directly at Layer 2:

1. The sender resolves the target's MAC address using an NDP Neighbor
   Solicitation ("who has address X?").
2. The switch floods the request, matched by `l2_ternary_table`, and the target
   replies with an NDP Neighbor Advertisement.
3. The sender transmits the frame, and `l2_exact_table` matches the
   destination MAC and forwards it to the correct port.

No IP-layer logic is required. The switch only moves frames between MAC
addresses it has already learned.

---

## Clarifying the Central Question
## The Decision Occurs Inside h2, Before Any Packet Is Sent

When h2 attempts to reach `2001:2:3::1` (h3), its kernel performs a
longest-prefix match against its own routing table:

| Destination | Within `2001:1:2::/64`? | Resulting action |
|---|---|---|
| `2001:2:3::1` (h3) | No | Forward to default gateway `2001:1:2::ff` |

Consequently, h2 never resolves h3's MAC address directly. Instead it
resolves the gateway's address and delegates the cross-subnet forwarding
problem to the switch.

Note that a broadcast domain (Layer 2) and a subnet (Layer 3) are
independent concepts. In this topology each host resides on its own link,
so the separation is logical and defined entirely in host configuration.

---

## Two Reasons Cross-Subnet Ping Fails Initially

**Problem A: the switch cannot answer for its gateway address.**
h2 issues an NDP Neighbor Solicitation for `2001:1:2::ff`, but the switch
has no networking stack and therefore does not reply. The neighbor table
shows the entry as `FAILED`, and the packet never leaves h2.

**Problem B: the switch cannot route by IP address.**
Even if h2 could deliver the frame, there is no routing table. The switch
has no means to map `2001:2:3::/64` to a next hop, no MAC rewrite logic, and
no hop-limit handling.

Exercise 5 addresses both problems.

---

## The Solution

## P4: Create ndp_reply Table

This table allows the switch to generate its own NDP Neighbor Advertisement
replies directly in the data plane.

```p4
table ndp_reply {
    key     = { hdr.ndp.target_ipv6_addr : exact; }
    actions = { ndp_ns_to_na; @defaultonly NoAction; }
    ...
}
```

ONOS installs one entry per switch gateway address, mapping it to that
device's MAC address. A table hit indicates that the Solicitation
targets one of the switch's own addresses, and the `ndp_ns_to_na` action
transforms the Solicitation into an Advertisement carrying switch's MAC address.
No control-plane round trip is required.

---

## P4: my_station Table

This table ensures that only frames addressed to the router's MAC are
routed.

```p4
table l2_my_station {
    key     = { hdr.ethernet.dst_addr : exact; }
    actions = { @defaultonly NoAction; }   // carries no data; the hit is the signal
}
```

The `.hit` result answers the question "is this frame addressed to me as a
router?" The table stores no value, because the value that would be needed
(`myStationMac`) is already present in `eth.dst`. Indeed, that is the reason
the table matched.

---

## P4: l3_ipv6_routing Table with ECMP

```p4
table l3_ipv6_routing {
    key = {
        hdr.ipv6.dst_addr   : lpm;        // selects the prefix and ECMP group
        hdr.ipv6.src_addr   : selector;   // hashed to choose a next hop
        hdr.ipv6.dst_addr   : selector;
        hdr.ipv6.flow_label : selector;
    }
    actions        = { routing; @defaultonly NoAction; }
    implementation = ecmp_selector;
}
```

The longest-prefix match selects the route. The action selector hashes the
flow identifiers (source, destination, and flow label, per [RFC 6438](https://datatracker.ietf.org/doc/html/rfc6438)) to
choose one spine. Packets of the same flow follow the same path, avoiding
reordering, while different flows are distributed across the spines.

### routing Action

```p4
action routing(mac_addr_t next_hop) {
    hdr.ethernet.src_addr = hdr.ethernet.dst_addr; // former dst (myStationMac) becomes src
    hdr.ethernet.dst_addr = next_hop;              // dst becomes the next hop
    hdr.ipv6.hop_limit    = hdr.ipv6.hop_limit - 1;
}
```

---

## Integration into the Pipeline (apply Block)

```p4
if (hdr.icmpv6.isValid() && hdr.icmpv6.type == ICMP6_TYPE_NS) {
    if (ndp_reply.apply().hit) { do_l3_l2 = false; } // replied, so skip L2 and L3
}
if (do_l3_l2) {
    if (l2_my_station.apply().hit) {                 // addressed to me as a router?
        if (l3_ipv6_routing.apply().hit) {           // route and rewrite MACs
            if (hdr.ipv6.hop_limit == 0) { drop(); exit; } // expired, so stop
        }
    }
    if (!l2_exact_table.apply().hit) { l2_ternary_table.apply(); } // deliver
}
```

The ordering is deliberate: NDP handling first, then L3 before L2, with an
`exit` so that a dropped packet cannot be re-forwarded by the subsequent L2
lookup.

---

## Control Plane: ONOS Application

**`NdpReplyComponent.java`** populates the `ndp_reply` table from
`netcfg.json`, mapping each gateway IPv6 address to `myStationMac` on every
device.

**`Ipv6RoutingComponent.java`** implements four methods:

| Method | Responsibility |
|---|---|
| `setUpMyStationTable()` | Installs the `l2_my_station` entry for the device MAC |
| `createNextHopGroup()` | Builds the ECMP SELECT group (`ecmp_selector`) |
| `createRoutingRule()` | Installs the LPM route pointing at the group |
| `createL2NextHopRule()` | Maps a next-hop MAC to an output port (`l2_exact_table`) |

Both components are activated by setting `enabled = true`.

---

## Verification

- `make p4-test TEST=routing` exercises the pipeline through the PTF tests.
- Regression checks: `TEST=packetio` and `TEST=bridging`.
- In Mininet, `h2 ping h3` followed by `h3 ping h2` produces replies with a
  decremented hop limit.
- `h3 ip -6 n` shows the gateway as `REACHABLE` with `myStationMac`,
  confirming P4-based NDP Advertisement generation.
- An iperf run with five parallel flows, observed in the ONOS user
  interface, shows traffic distributed across both spines, confirming ECMP.

---

## Summary

- The fabric was extended from L2 bridging (forwarding by MAC address) to
  L3 routing (forwarding by IP prefix, hop by hop, with MAC rewriting at
  each hop).
- Hosts already possessed knowledge of their own subnets through their
  configuration. The switch needed only to assume the roles of gateway and
  router.
- The solution comprises three P4 tables (NDP reply, My Station, and IPv6
  routing with ECMP) and one routing action, driven by two ONOS components
  that read `netcfg.json`.

The result is full connectivity between any pair of hosts, load balanced
across the spine switches.
