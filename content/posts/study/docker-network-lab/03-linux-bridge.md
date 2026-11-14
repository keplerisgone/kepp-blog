---
title: "Docker 네트워크 실험 03 — Linux Bridge와 Layer 2 Forwarding"
date: "2026-11-14T19:25:40+09:00"
draft: false
description: "세 개의 Network Namespace를 Linux Bridge로 연결하고 Layer 2 프레임 전달과 MAC 주소 학습을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Networking", "Network Namespace", "veth", "Bridge"]
ShowToc: true
---

## 1. Objective

Understand how a Linux Bridge connects multiple isolated Network Namespaces and forwards Ethernet frames at Layer 2.

Success criteria:
- Connect three Network Namespaces through one Linux Bridge.
- Establish communication between all connected namespaces.
- Observe MAC address learning through the Forwarding Database (FDB).

## 2. Network Topology

```text
                   br-lab
                 /    |    \
          veth1-br veth2-br veth3-br
              |       |       |
          veth1-ns veth2-ns veth3-ns
              |       |       |
             lab1    lab2    lab3
              |       |       |
          10.10.0.1  .0.2    .0.3
                Subnet: /24
```

Each namespace is connected to `br-lab` through a separate veth pair.

The bridge operates at Layer 2, forwarding Ethernet frames according to destination MAC addresses.

## 3. Experiment

### 3.1 Create the Bridge and Namespaces

```bash
sudo ip link add br-lab type bridge
sudo ip link set br-lab up

sudo ip netns add lab1
sudo ip netns add lab2
sudo ip netns add lab3
```

### 3.2 Create and Attach veth Pairs

```bash
sudo ip link add veth1-br type veth peer name veth1-ns
sudo ip link add veth2-br type veth peer name veth2-ns
sudo ip link add veth3-br type veth peer name veth3-ns

sudo ip link set veth1-ns netns lab1
sudo ip link set veth2-ns netns lab2
sudo ip link set veth3-ns netns lab3

sudo ip link set veth1-br master br-lab
sudo ip link set veth2-br master br-lab
sudo ip link set veth3-br master br-lab

sudo ip link set veth1-br up
sudo ip link set veth2-br up
sudo ip link set veth3-br up
```

**Observation:**

All three host-side veth endpoints were attached to `br-lab`.

Initially, the bridge reported `NO-CARRIER` and its ports were in the `disabled` state because the namespace-side endpoints were not operational.

### 3.3 Configure IP Addresses

```bash
sudo ip netns exec lab1 ip addr add 10.10.0.1/24 dev veth1-ns
sudo ip netns exec lab2 ip addr add 10.10.0.2/24 dev veth2-ns
sudo ip netns exec lab3 ip addr add 10.10.0.3/24 dev veth3-ns

sudo ip netns exec lab1 ip link set veth1-ns up
sudo ip netns exec lab2 ip link set veth2-ns up
sudo ip netns exec lab3 ip link set veth3-ns up

sudo ip netns exec lab1 ip link set lo up
sudo ip netns exec lab2 ip link set lo up
sudo ip netns exec lab3 ip link set lo up
```

**Observation:**

After activating the namespace-side endpoints:
- `br-lab` entered the `UP` state.
- All three bridge ports entered the `forwarding` state.
- All three namespaces were configured in `10.10.0.0/24`.

No IP address was assigned to `br-lab` because the bridge was only needed for Layer 2 forwarding.

### 3.4 Verify Connectivity

```bash
sudo ip netns exec lab1 ping -c 3 10.10.0.2
sudo ip netns exec lab1 ping -c 3 10.10.0.3
```

**Results:**

| Test | Transmitted | Received | Packet Loss |
|---|---:|---:|---:|
| lab1 → lab2 | 3 | 3 | 0% |
| lab1 → lab3 | 3 | 3 | 0% |

All three namespaces were able to communicate through the Linux Bridge.

No router or IP forwarding configuration was required.

### 3.5 Inspect the Forwarding Database

```bash
bridge fdb show br br-lab
```

Selected dynamically learned entries:

```text
1e:b9:97:a7:eb:6a dev veth1-br master br-lab
52:44:9b:64:7b:d2 dev veth2-br master br-lab
c6:18:0c:97:e7:e0 dev veth3-br master br-lab
```

These entries demonstrate that the bridge learned which MAC addresses were reachable through specific ports.

The output also contained permanent multicast entries:

```text
33:33:00:00:00:01 dev veth1-br self permanent
01:00:5e:00:00:01 dev veth1-br self permanent
33:33:ff:b3:94:23 dev veth1-br self permanent
33:33:00:00:00:fb dev veth1-br self permanent
```

These are multicast-related local entries, not four additional network devices.

## 4. Analysis

### Layer 2 Forwarding

A Linux Bridge acts as a software Ethernet switch.

It examines the destination MAC address of incoming Ethernet frames and uses its FDB to determine the appropriate outgoing port.

### MAC Address Learning

The bridge learns MAC-to-port mappings from the source MAC addresses of incoming frames.

- Known unicast: Forward to the learned destination port.
- Unknown unicast: Typically flood to other eligible ports.
- Broadcast: Flood to other eligible ports.

For example, an ARP Request from `lab1` is normally broadcast to both `lab2` and `lab3`, even when the requested IPv4 address belongs only to `lab2`.

Only the namespace owning the requested address should respond under ordinary conditions.

### Switching vs. Routing

A bridge primarily forwards Ethernet frames based on MAC addresses.

Unlike a router, it does not require an IP address or an IP routing table to perform ordinary Layer 2 forwarding.

All three namespaces were configured in the same IPv4 subnet to avoid requiring Layer 3 routing.

## 5. Conclusion

Successfully constructed a Linux Bridge connecting three isolated Network Namespaces.

Verified:
1. Bridge port configuration and operational state changes.
2. ICMP connectivity between namespaces.
3. MAC address learning through the FDB.
4. The distinction between permanent multicast entries and learned MAC entries.

**Key finding:** A Linux Bridge provides Layer 2 connectivity between separate network namespaces without requiring a router.

## 6. Further Investigation

- Capture ARP broadcast flooding on each bridge port.
- Compare known-unicast forwarding with unknown-unicast flooding.
- Investigate VLAN-aware bridging.
- Compare this topology with Docker's default bridge network.
