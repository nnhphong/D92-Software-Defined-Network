## Myths
### 1. Why does `h1b ping -c 1 h2` never works:
    - `h1b` realize itself and `h2` is not the same LAN
    - `h1b` find MAC address of router through `NS` message
    - router replies with solicated `NA` message, update `h1b` cache to `REACHABLE`
    - `h1b` sends packet with `h2`'s IP and router's MAC address
    - Now router need `h2`'s MAC. **BUT** it doesn't have capability to resolve `h2`'s MAC in the first place. That is the job of the **controller** to detect hosts when it starts to send some traffic.

Therefore, every host must "warm up" by sending some traffic to get detected by ONOS before it can able to ping each other.

### 2. `h1b ping -c 1 h1c` packet loss for the first time
Scenario:

    mininet> h1a ping -c 1 h1c
    PING 2001:1:1::c(2001:1:1::c) 56 data bytes

    --- 2001:1:1::c ping statistics ---
    1 packets transmitted, 0 received, 100% packet loss, time 0ms

    mininet> h1a ip -6 neigh show
    2001:1:1::c dev h1a-eth0 lladdr 00:00:00:00:00:1c REACHABLE
    2001:1:1::b dev h1a-eth0 lladdr 00:00:00:00:00:1b STALE
    mininet> h1a ping -c 1 h1c
    PING 2001:1:1::c(2001:1:1::c) 56 data bytes
    64 bytes from 2001:1:1::c: icmp_seq=1 ttl=64 time=5.41 ms

    --- 2001:1:1::c ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms

Reasons: flow rule take time to respond and take effective. While flow rule hasn't installed, router doesn't know where to forward this ping packet -> packet dropped.

### 3. Why does NDP reply use `IPV6_MCAST_01`
- They send NDP reply back using single port anyway (I guess it will override multicasting) 
    `standard_metadata.egress_spec = standard_metadata.ingress_port;`
- So putting destination address as `IPV6_MCAST_01` is a lazy way of doing it
- I changed ethernet `dst_addr` assign to ethernet `src_addr` and working fine.

### 4. `STALE` -> `FAILED` h1b's NDP cache

(unresolved)

How NDP works: https://oneuptime.com/blog/post/2026-03-20-ndp-nud-states/view#probe-state 

# Exercise 6: Segment Routing (SRv6)
## SRv6

The IPv6 routing header looks as follows:
```
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Next Header   |  Hdr Ext Len  | Routing Type  | Segments Left |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  Last Entry   |     Flags     |              Tag              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |            Segment List[0] (128 bits IPv6 address)            |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |                                                               |
                                  ...
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                                                               |
    |            Segment List[n] (128 bits IPv6 address)            |
    |                                                               |
    |                                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
