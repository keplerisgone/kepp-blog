---
title: "Docker 네트워크 실험 02 — Network Namespace와 veth"
date: "2026-11-11T00:00:00+09:00"
draft: true
description: "두 Network Namespace를 veth pair로 연결하고 ICMP 통신과 이웃 주소 해석을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Networking", "Network Namespace", "veth"]
ShowToc: true
---

## Overview

Investigate Linux Network Namespace isolation and establish communication between two isolated networking environments using a virtual Ethernet pair.

Success criteria:

- Create two independent Network Namespaces.
- Verify that their network resources are isolated from the host.
- Connect the namespaces using a veth pair.
- Establish bidirectional-capable IP connectivity and verify it with ICMP.
- Confirm neighbor resolution.

## Background

```text
                 Linux Kernel
┌───────────────────────────────────────────────┐
│                                               │
│   Network Namespace ns1    Namespace ns2      │
│   ┌─────────────────┐    ┌─────────────────┐  │
│   │                 │    │                 │  │
│   │ veth1           │    │ veth2           │  │
│   │ 10.10.0.1/24    │    │ 10.10.0.2/24    │  │
│   │      │          │    │      │          │  │
│   └──────┼──────────┘    └──────┼──────────┘  │
│          └──────── veth pair ───┘             │
│                                               │
└───────────────────────────────────────────────┘
```

Both namespaces share the same Linux kernel but maintain separate networking resources.

No physical NIC, bridge, router, or NAT was required for this experiment.

## Environment

- OS: Arch Linux (Omarchy)
- Tools: iproute2 (`ip`), iputils (`ping`)
- Topology: Network Namespaces `ns1` and `ns2`, connected by a veth pair

## Implementation / Experiment

### Experiment

#### 3.1 Create Network Namespaces

```bash
sudo ip netns add ns1
sudo ip netns add ns2

ip netns list
sudo ip netns exec ns1 ip -br link
sudo ip netns exec ns2 ip -br link
```

**Observation:**

Both namespaces were successfully created, and each initially contained only a loopback interface in the `DOWN` state.

Further inspection showed that `ns1` initially had an empty IPv4 routing table.

The host retained its existing interfaces and routes, including `wlo1` and `docker0`.

**Analysis:**

A Network Namespace provides an isolated instance of Linux networking resources, including interfaces, IP addresses, routing tables, and neighbor tables.

Creating a namespace does not automatically provide connectivity to other namespaces or to the host network.

#### 3.2 Create a veth Pair

```bash
sudo ip link add veth1 type veth peer name veth2

ip -br link show veth1
ip -br link show veth2
```

**Observation:**

Two virtual Ethernet interfaces were created in the host network namespace.

Both initially had a `DOWN` operational state.

**Analysis:**

A veth pair consists of two connected virtual Ethernet interfaces.

Frames transmitted through one endpoint are delivered to the other endpoint.

Unlike two independent virtual interfaces, the pair is connected at creation.

#### 3.3 Move Interfaces into Namespaces

```bash
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2
```

Verification:

```bash
sudo ip netns exec ns1 ip -br link
sudo ip netns exec ns2 ip -br link
```

**Observation:**

- `veth1` appeared inside `ns1`.
- `veth2` appeared inside `ns2`.
- The original veth pair remained connected across the namespace boundary.

**Analysis:**

Moving an interface into a namespace changes its network namespace membership.

The interface is not recreated; the existing interface becomes visible and configurable within its destination namespace.

This allows communication between otherwise isolated networking environments.

#### 3.4 Configure IP Addresses

```bash
sudo ip netns exec ns1 ip addr add 10.10.0.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 10.10.0.2/24 dev veth2

sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up

sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns2 ip link set lo up
```

Verification:

```bash
sudo ip netns exec ns1 ip -br addr
sudo ip netns exec ns2 ip -br addr
```

**Observation:**

Both interfaces became `UP` and received their assigned IPv4 addresses.

| Namespace | Interface | IP Address |
|---|---|---|
| ns1 | veth1 | 10.10.0.1/24 |
| ns2 | veth2 | 10.10.0.2/24 |

Both addresses belong to the same subnet: `10.10.0.0/24`.

#### 3.5 Verify Connectivity

```bash
sudo ip netns exec ns1 ping -c 3 10.10.0.2
```

**Observed result:**

```text
64 bytes from 10.10.0.2: icmp_seq=1 ttl=64 time=0.033 ms
64 bytes from 10.10.0.2: icmp_seq=2 ttl=64 time=0.062 ms
64 bytes from 10.10.0.2: icmp_seq=3 ttl=64 time=0.030 ms

3 packets transmitted, 3 received, 0% packet loss
```

**Analysis:**

Successful ICMP communication confirms that packets can travel between the namespaces through the veth pair.

Because both interfaces belong to the same directly connected subnet, no additional router is necessary.

#### 3.6 Verify Neighbor Resolution

```bash
sudo ip netns exec ns1 ip neigh
```

**Observed result:**

```text
10.10.0.2 dev veth1 lladdr 46:5e:7c:e9:60:64 REACHABLE
```

**Analysis:**

`ns1` successfully resolved the link-layer address associated with `10.10.0.2`.

This demonstrates that neighbor resolution operates inside the isolated networking environment.

The result is consistent with successful ARP resolution, although ARP packets were not directly captured on the veth interface during this experiment.

## Results

Successfully constructed a minimal isolated Linux network using two Network Namespaces and a veth pair.

The experiment demonstrated that separate networking environments can communicate through an explicitly configured virtual link while sharing a single Linux kernel.

This provides a foundation for understanding Docker container networking, where Network Namespaces and veth pairs are commonly combined with Linux Bridges, routing, and NAT.

## Problems and Solutions

New namespaces initially had only a loopback interface in the `DOWN` state and no connectivity. The experiment explicitly connected them with a veth pair, assigned IP addresses, and brought the interfaces up.

## Lessons Learned

1. **Network Namespace Isolation:** Each namespace has its own network interfaces and routing configuration.
2. **veth Pair:** Two virtual Ethernet interfaces can remain connected even when assigned to different namespaces.
3. **IP Configuration:** Each namespace can independently configure its interfaces and IP addresses.
4. **Connectivity:** Network isolation does not inherently prohibit communication. Connectivity can be explicitly established.
5. **Neighbor Resolution:** A namespace can resolve the MAC address of a peer on its directly connected network.

### Further Investigation

- Capture ARP and ICMP packets directly on veth interfaces.
- Inspect each namespace's connected route.
- Connect multiple namespaces using a Linux Bridge.
- Compare the manually constructed topology with Docker Bridge Networking.

## References

No external references were recorded in the source notes.

