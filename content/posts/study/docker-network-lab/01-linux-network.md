---
title: "Docker 네트워크 실험 01 — Linux 네트워크 기초"
date: "2026-11-11T00:00:00+09:00"
draft: true
description: "라우팅 조회, ARP와 Neighbor Table, ICMP 패킷 캡처 실험 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Networking", "Routing", "ARP", "ICMP"]
ShowToc: true
---

## Overview

Understand how Linux selects network routes, resolves link-layer addresses, and exchanges IP packets.

## Background

This experiment examines routing, ARP, the Linux Neighbor Table, and ICMP on the host network.

## Environment

- OS: Arch Linux (Omarchy)
- Tools: iproute2 (`ip`), iputils (`ping`)
- Packet capture: tcpdump
- Network: Wi-Fi (`wlo1`), Docker Bridge (`docker0`)

## Implementation / Experiment

### Routing Experiment

Commands:

```bash
ip route get 172.30.1.254
ip route get 8.8.8.8
ip route get 172.17.0.2
```

#### Observations

| Destination | Interface | Next Hop | Source IP |
|---|---|---|---|
| 172.30.1.254 | wlo1 | Direct | 172.30.1.8 |
| 8.8.8.8 | wlo1 | 172.30.1.254 | 172.30.1.8 |
| 172.17.0.2 | docker0 | Direct | 172.17.0.1 |

#### Analysis

Linux selects a route according to its routing configuration, typically choosing the most specific matching prefix.

- `dev` identifies the outgoing interface.
- `via` identifies the next-hop router.
- `src` identifies the selected source IP address.

A successful route lookup does not necessarily guarantee packet delivery.

### ARP Experiment

Commands:

```bash
ip neigh show dev wlo1
sudo tcpdump -eni wlo1 'arp'
```

#### Observations

The gateway `172.30.1.254` appeared in the neighbor table with a MAC address and the `REACHABLE` state.

Captured traffic included ARP requests and replies between hosts on the local network.

Example from the capture:

```text
Request who-has 172.30.1.74 tell 172.30.1.8
Reply 172.30.1.74 is-at e2:1b:25:e6:5b:93
```

#### Analysis

ARP resolves an IPv4 address to a link-layer address on an ARP-capable network.

Linux stores resolved addresses in its Neighbor Table, reducing the need for repeated ARP requests.

The capture demonstrated address resolution, although not every captured ARP exchange was caused by our test commands.

### ICMP Experiment

Commands:

```bash
sudo tcpdump -ni wlo1 'icmp and host 172.30.1.254'
ping -c 3 172.30.1.254
```

#### Observations

```text
172.30.1.8 > 172.30.1.254: ICMP echo request, seq 1
172.30.1.254 > 172.30.1.8: ICMP echo reply, seq 1
```

Three Echo Requests and three corresponding Echo Replies were captured.

**Result:** 3 packets transmitted, 3 received, 0% packet loss.

#### Analysis

The experiment confirmed bidirectional ICMP communication between the host and its default gateway.

Routing determines the outgoing path, while neighbor resolution provides the link-layer address needed for local delivery.

## Results

**Result:** 3 packets transmitted, 3 received, 0% packet loss when pinging the default gateway. Routing decisions and ARP exchanges were also observed as recorded above.

## Problems and Solutions

No specific failure or corrective procedure was recorded in the source notes.

## Lessons Learned

- Verified Linux routing decisions using `ip route get`.
- Observed IPv4-to-MAC address resolution using ARP.
- Inspected the Linux Neighbor Table.
- Captured ICMP Echo Requests and Replies using tcpdump.

**Remaining:** An active TCP/UDP socket-binding experiment was not performed. Socket inspection was completed previously using `ss`.

## References

No external references were recorded in the source notes.

