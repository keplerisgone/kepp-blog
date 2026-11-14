---
title: "Docker 네트워크 실험 05 — NAT와 Packet Translation"
date: "2026-11-14T19:29:56+09:00"
draft: false
description: "nftables로 Source NAT를 구성하고 패킷 캡처를 통해 주소 변환과 연결 추적을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Networking", "NAT", "nftables", "Conntrack"]
ShowToc: true
---

## 1. Objective

Understand how Network Address Translation modifies packet addresses and verify Source NAT using nftables and packet captures.

Success criteria:
- Configure a Source NAT rule.
- Verify that source IP addresses change across the router.
- Maintain successful communication after translation.
- Compare packet captures before and after NAT.

## 2. Network Topology

Reuse the routing topology from Experiment 04.

```text
route-a                  route-r                  route-b
10.10.1.2               NAT Router               10.10.2.2
    |                       |                         |
    |                   10.10.1.1                     |
    |                   10.10.2.1                     |
    +-----------------------+-------------------------+

Before NAT:
10.10.1.2 → 10.10.2.2

After SNAT:
10.10.2.1 → 10.10.2.2
```

The NAT configuration is applied inside `route-r`, leaving the host's existing Docker, UFW, and Tailscale firewall tables unchanged.

## 3. Experiment

### 3.1 Verify IP Forwarding

```bash
sudo ip netns exec route-r sysctl net.ipv4.ip_forward
```

**Observed result:**

```text
net.ipv4.ip_forward = 1
```

Forwarding was enabled during the NAT experiment.

### 3.2 Capture Packets Before NAT

Run:

```bash
sudo ip netns exec route-r tcpdump -ni veth-rb icmp
```

Generate traffic:

```bash
sudo ip netns exec route-a ping -c 3 10.10.2.2
```

**Observation:**

```text
IP 10.10.1.2 > 10.10.2.2: ICMP echo request
IP 10.10.2.2 > 10.10.1.2: ICMP echo reply
```

The source and destination IP addresses were unchanged during routing.

### 3.3 Configure Source NAT

Create a dedicated NAT table:

```bash
sudo ip netns exec route-r nft add table ip lab_nat
```

Create a postrouting NAT chain:

```bash
sudo ip netns exec route-r nft \
  'add chain ip lab_nat postrouting { type nat hook postrouting priority srcnat; policy accept; }'
```

Add a masquerading rule:

```bash
sudo ip netns exec route-r nft \
  'add rule ip lab_nat postrouting ip saddr 10.10.1.0/24 oifname "veth-rb" masquerade'
```

Verify:

```bash
sudo ip netns exec route-r nft list table ip lab_nat
```

**Observed configuration:**

```nft
table ip lab_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.10.1.0/24 oifname "veth-rb" masquerade
    }
}
```

### 3.4 Verify Connectivity After NAT

```bash
sudo ip netns exec route-a ping -c 3 10.10.2.2
```

**Observed result:**

```text
3 packets transmitted, 3 received, 0% packet loss
```

Communication remained functional after enabling Source NAT.

### 3.5 Capture Packets After NAT

```bash
sudo ip netns exec route-r tcpdump -ni veth-rb icmp
```

Generate ICMP traffic from `route-a`.

**Observed packet capture:**

```text
IP 10.10.2.1 > 10.10.2.2: ICMP echo request
IP 10.10.2.2 > 10.10.2.1: ICMP echo reply
```

The source address of outgoing Echo Requests changed from `10.10.1.2` to `10.10.2.1`.

This is direct evidence of Source NAT.

## 4. Analysis

### Source NAT

Source NAT modifies the source IP address of matching outgoing packets.

The masquerading rule used the IP address assigned to the outgoing interface `veth-rb`, which was `10.10.2.1`.

| Packet field | Before NAT | After NAT |
|---|---|---|
| Source IP | 10.10.1.2 | 10.10.2.1 |
| Destination IP | 10.10.2.2 | 10.10.2.2 |

### Connection Tracking

The destination namespace sends replies to the translated source address, `10.10.2.1`.

Linux connection tracking maintains state associated with the translation, allowing the router to reverse the address mapping on replies.

The successful ping and post-NAT capture are consistent with this behavior.

The connection-tracking table itself was not directly inspected.

### Why NAT Is Useful

Private IPv4 addresses are not globally routed on the public internet.

Source NAT allows packets originating from private networks to use another address when communicating with external networks.

Common applications include:
- Home routers sharing an outward-facing IPv4 address among multiple devices.
- Docker containers accessing external networks through a host.
- Network environments requiring address translation between different administrative domains.

However, NAT is not inherently required for communication between all different IP subnets.

The previous routing experiment demonstrated that two private subnets could communicate without NAT when appropriate routes and forwarding were configured.

## 5. Routing vs. NAT

| Mechanism | Purpose |
|---|---|
| Switching | Forward Ethernet frames using MAC addresses |
| Routing | Forward IP packets according to routing tables |
| NAT | Translate source or destination address information |

Routing and NAT are separate mechanisms that can be used together.

In this experiment, `route-r` performed both IP forwarding and Source NAT.

## 6. Conclusion

Successfully configured Source NAT using nftables in an isolated Network Namespace.

Verified:
1. IPv4 forwarding was enabled.
2. The masquerading rule was installed.
3. Source IP translation occurred on the outgoing interface.
4. ICMP communication continued to succeed after translation.
5. The translated source address was visible in tcpdump.

**Key finding:** NAT allows a router to translate packet addresses while maintaining communication through connection-tracking state.

## 7. Cleanup

To remove only the custom NAT table:

```bash
sudo ip netns exec route-r nft delete table ip lab_nat
```

This command was recommended but its execution has not been verified.

## 8. Further Investigation

- Inspect connection-tracking entries using `conntrack`.
- Compare packet captures on both sides of a NAT router.
- Implement Destination NAT and port forwarding.
- Inspect Docker's automatically generated masquerading rules.
- Compare manual NAT configuration with Docker Bridge Networking.
