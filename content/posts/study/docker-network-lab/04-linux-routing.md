---
title: "Docker 네트워크 실험 04 — Linux Routing과 IP Forwarding"
date: "2026-11-14T19:29:15+09:00"
draft: false
description: "서로 다른 두 서브넷 사이에 Linux 라우터를 구성하고 정적 경로와 IP Forwarding을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Networking", "Routing", "IP Forwarding", "ICMP"]
ShowToc: true
---

## 1. Objective

Understand how Linux forwards IPv4 packets between different subnets.

Success criteria:
- Construct two separate IPv4 networks.
- Configure a Linux Network Namespace as a router.
- Reproduce communication failure without valid routes.
- Compare communication with IP forwarding enabled and disabled.
- Capture packets crossing the router.

## 2. Network Topology

```text
    Subnet A                  Subnet B
  10.10.1.0/24              10.10.2.0/24

   route-a                   route-b
  10.10.1.2                 10.10.2.2
      |                         |
    veth-a                    veth-b
      |                         |
    veth-ra                  veth-rb
      |                         |
      +--------- route-r -------+
              Linux Router

          10.10.1.1 / 10.10.2.1
```

The router namespace contains two interfaces, each connected to a different subnet.

Unlike the previous experiment, no Linux Bridge is used.

## 3. Experiment

### 3.1 Create Namespaces and veth Pairs

```bash
sudo ip netns add route-a
sudo ip netns add route-r
sudo ip netns add route-b

sudo ip link add veth-a type veth peer name veth-ra
sudo ip link add veth-b type veth peer name veth-rb

sudo ip link set veth-a netns route-a
sudo ip link set veth-ra netns route-r
sudo ip link set veth-b netns route-b
sudo ip link set veth-rb netns route-r
```

### 3.2 Configure Interfaces

```bash
sudo ip -n route-a addr add 10.10.1.2/24 dev veth-a
sudo ip -n route-r addr add 10.10.1.1/24 dev veth-ra
sudo ip -n route-r addr add 10.10.2.1/24 dev veth-rb
sudo ip -n route-b addr add 10.10.2.2/24 dev veth-b

sudo ip -n route-a link set veth-a up
sudo ip -n route-r link set veth-ra up
sudo ip -n route-r link set veth-rb up
sudo ip -n route-b link set veth-b up
```

### 3.3 Inspect Routing Tables

```bash
sudo ip -n route-a route
sudo ip -n route-r route
sudo ip -n route-b route
```

**Observation:**

| Namespace | Connected routes |
|---|---|
| route-a | 10.10.1.0/24 |
| route-r | 10.10.1.0/24, 10.10.2.0/24 |
| route-b | 10.10.2.0/24 |

The router namespace had routes to both networks because it was directly connected to both.

However, the endpoint namespaces initially knew only their respective local subnets.

### 3.4 Test Communication Without Static Routes

```bash
sudo ip netns exec route-a ping -c 3 10.10.2.2
```

**Observed result:**

```text
ping: connect: Network is unreachable
```

**Analysis:**

`route-a` had no route matching the destination subnet.

The kernel could not select an outgoing route for `10.10.2.2`.

### 3.5 Configure Static Routes

```bash
sudo ip -n route-a route add 10.10.2.0/24 via 10.10.1.1
sudo ip -n route-b route add 10.10.1.0/24 via 10.10.2.1
```

Verify:

```bash
sudo ip -n route-a route get 10.10.2.2
sudo ip -n route-b route get 10.10.1.2
```

**Observed results:**

```text
10.10.2.2 via 10.10.1.1 dev veth-a src 10.10.1.2
10.10.1.2 via 10.10.2.1 dev veth-b src 10.10.2.2
```

Each endpoint now had a route to the other subnet through the router.

### 3.6 Enable and Disable IP Forwarding

Check the forwarding state:

```bash
sudo ip netns exec route-r sysctl net.ipv4.ip_forward
```

The initial value was:

```text
net.ipv4.ip_forward = 1
```

With forwarding enabled, the ping succeeded:

```text
3 packets transmitted, 3 received, 0% packet loss
```

The received ICMP replies reported `ttl=63`.

Disable forwarding:

```bash
sudo ip netns exec route-r sysctl -w net.ipv4.ip_forward=0
```

Repeat the ping:

```bash
sudo ip netns exec route-a ping -c 3 10.10.2.2
```

**Observed result:**

```text
3 packets transmitted, 0 received, 100% packet loss
```

Restore forwarding:

```bash
sudo ip netns exec route-r sysctl -w net.ipv4.ip_forward=1
```

### 3.7 Capture Forwarded Packets

```bash
sudo ip netns exec route-r tcpdump -ni veth-ra icmp
```

Generate traffic:

```bash
sudo ip netns exec route-a ping -c 3 10.10.2.2
```

The capture showed:

```text
IP 10.10.1.2 > 10.10.2.2: ICMP echo request
IP 10.10.2.2 > 10.10.1.2: ICMP echo reply
```

The same source and destination IP addresses were later observed on the router's `veth-rb` interface before NAT was configured.

## 4. Analysis

### Routing vs. IP Forwarding

Routing tables determine which next hop and outgoing interface should be selected for a destination.

IP forwarding determines whether Linux can forward a packet received from another system.

Both correct routing information and enabled forwarding were necessary for this experiment.

### Layer 2 vs. Layer 3

The router forwards IP packets between different subnets.

During forwarding:
- The source and destination IP addresses normally remain unchanged.
- New link-layer frames are constructed for the outgoing interface.
- The IPv4 TTL decreases by one.

In the experiment, `ttl=63` was consistent with an ICMP reply originating with TTL 64 and passing through one router.

### Isolation

The forwarding setting was changed inside `route-r`, not in the host's network namespace.

This allowed the experiment to avoid changing the host's forwarding configuration.

## 5. Conclusion

Successfully configured Linux to route IPv4 packets between two isolated subnets.

Verified three distinct outcomes:

| Condition | Result |
|---|---|
| Missing route | Network is unreachable |
| Routes present, forwarding enabled | Communication succeeds |
| Routes present, forwarding disabled | Communication fails |

**Key finding:** A routing table determines where packets should go, while IP forwarding controls whether the kernel forwards packets between interfaces.

## 6. Further Investigation

- Inspect ICMP Time Exceeded packets when TTL expires.
- Compare Ethernet headers before and after routing.
- Examine Linux policy routing.
- Investigate forwarding-related firewall rules.

