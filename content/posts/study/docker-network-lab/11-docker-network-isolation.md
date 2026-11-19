---
title: "Docker 네트워크 실험 11 — Docker Bridge 네트워크 격리"
date: "2026-11-19T00:00:00+09:00"
draft: false
description: "서로 다른 Docker bridge 네트워크의 통신 격리를 재현하고 게이트웨이 연결과 forwarding 및 방화벽 규칙을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Networking", "Linux Bridge", "Network Isolation", "iptables"]
ShowToc: true
---

## 1. Objective

Investigate how Docker isolates containers connected to separate user-defined bridge networks.

Compare communication within an individual bridge network with communication across independent Docker networks.

**Success Criteria**
- Create two independent Docker bridge networks.
- Configure containers with addresses in different subnets.
- Verify each container can communicate with its gateway.
- Reproduce communication failure between separate Docker networks.
- Inspect Docker's forwarding and filtering rules.
- Investigate multi-homed container connectivity.

## 2. Network Topology

```text
          Linux Host
               |
       ┌───────┴───────┐
       |               |
lab-isolated-a    lab-isolated-b
10.81.0.0/24      10.82.0.0/24
       |               |
  isolated-a       isolated-b
  10.81.0.2        10.82.0.2
       |               |
       └── No direct ──┘
         communication
```

Each Docker network is implemented using its own bridge and subnet.

## 3. Experiment

### 3.1 Create Two Docker Networks

```bash
sudo docker network create \
  --driver bridge \
  --subnet 10.81.0.0/24 \
  lab-isolated-a

sudo docker network create \
  --driver bridge \
  --subnet 10.82.0.0/24 \
  lab-isolated-b
```

Both commands completed successfully.

Inspect network configuration:

```bash
sudo docker network inspect lab-isolated-a \
  --format '{{json .IPAM.Config}}'

sudo docker network inspect lab-isolated-b \
  --format '{{json .IPAM.Config}}'
```

**Observed results:**

```json
[{"Subnet":"10.81.0.0/24","Gateway":"10.81.0.1"}]
```

```json
[{"Subnet":"10.82.0.0/24","Gateway":"10.82.0.1"}]
```

Docker created independent bridge networks with separate IPv4 subnets and gateways.

### 3.2 Create Isolated Containers

```bash
sudo docker run -d \
  --name isolated-a \
  --network lab-isolated-a \
  --ip 10.81.0.2 \
  alpine:3.22 sleep 3600

sudo docker run -d \
  --name isolated-b \
  --network lab-isolated-b \
  --ip 10.82.0.2 \
  alpine:3.22 sleep 3600
```

Verify configuration:

```bash
sudo docker exec isolated-a ip addr
sudo docker exec isolated-b ip addr
```

**Observed addresses:**

| Container | Interface | IPv4 |
|---|---|---|
| isolated-a | eth0 | 10.81.0.2/24 |
| isolated-b | eth0 | 10.82.0.2/24 |

The containers were successfully assigned addresses from their respective networks.

### 3.3 Verify Gateway Connectivity

```bash
sudo docker exec isolated-a ping -c 3 10.81.0.1
sudo docker exec isolated-b ping -c 3 10.82.0.1
```

**Observed results:**

| Test | Transmitted | Received | Loss |
|---|---:|---:|---:|
| isolated-a → 10.81.0.1 | 3 | 3 | 0% |
| isolated-b → 10.82.0.1 | 3 | 3 | 0% |

Both containers could communicate with their own bridge gateways.

### 3.4 Test Communication Across Networks

```bash
sudo docker exec isolated-a ping -c 3 -W 2 10.82.0.2
```

**Observed result:**

```text
3 packets transmitted
0 packets received
100% packet loss
```

The source container could communicate with its local gateway but not with the container on the other bridge.

This result is consistent with Docker's default inter-network isolation.

### 3.5 Inspect Docker Firewall Rules

Commands:

```bash
sudo iptables -S DOCKER-FORWARD
sudo iptables -S DOCKER-ISOLATION-STAGE-1
sudo iptables -S DOCKER-ISOLATION-STAGE-2
```

**Observed DOCKER-FORWARD rules:**

```text
-N DOCKER-FORWARD
-A DOCKER-FORWARD -j DOCKER-CT
-A DOCKER-FORWARD -j DOCKER-INTERNAL
-A DOCKER-FORWARD -j DOCKER-BRIDGE
```

The chain also contained interface-specific ACCEPT rules for Docker bridges.

The legacy isolation-chain inspection commands returned:

```text
iptables: No chain/target/match by that name.
```

**Analysis**

Docker's firewall chain structure depends on its version and firewall backend.

The absence of older `DOCKER-ISOLATION-STAGE-*` chains does not mean that network isolation is disabled.

The observed forwarding rules were not sufficient by themselves to identify the exact rule responsible for dropping the cross-network ICMP traffic.

Further inspection of the current Docker chain structure and packet counters is required for a definitive firewall-level explanation.

## 4. Analysis

### Layer 2 Separation

The two Docker networks use separate Linux Bridges.

Ethernet frames are not ordinarily switched directly between these bridges.

This creates separate Layer 2 broadcast domains.

### Layer 3 Routing

Each network has its own gateway on the Linux host.

The host may possess routes to both subnets.

However, route availability does not automatically mean forwarding between the two Docker bridge networks is permitted.

### Docker Network Isolation

Docker configures firewall rules to control communication between different bridge networks.

In this experiment, ICMP communication within each network succeeded, while direct communication across the two networks failed.

This demonstrates effective network separation in the tested configuration.

### Network Isolation vs. Complete Security Boundary

Docker network isolation restricts ordinary direct communication between separate container networks.

It does not necessarily prevent every possible communication path.

Host-mediated traffic, published ports, additional network attachments, and firewall configuration can change which endpoints are reachable.

## 5. Planned Multi-Network Experiment

The next step is to create a third container connected to both networks.

```bash
sudo docker run -d \
  --name isolated-c \
  --network lab-isolated-a \
  alpine:3.22 sleep 3600

sudo docker network connect lab-isolated-b isolated-c
```

Inspect:

```bash
sudo docker exec isolated-c ip addr
sudo docker exec isolated-c ip route
```

Test:

```bash
sudo docker exec isolated-c ping -c 3 10.81.0.2
sudo docker exec isolated-c ping -c 3 10.82.0.2
```

**Status:** Planned, not yet verified through terminal output.

### Multi-homing vs. Routing

A multi-homed container has interfaces connected to multiple networks.

It can communicate directly with reachable endpoints on those networks.

However, it does not automatically act as a router between them.

For traffic from `isolated-a` to reach `isolated-b` through `isolated-c`, additional routing and forwarding configuration would be necessary, and Docker's network filtering would also need to be considered.

## 6. Conclusion

Successfully reproduced Docker network isolation using two independent user-defined bridge networks.

**Verified Findings**
1. Two Docker bridge networks were created with separate subnets.
2. Containers received the expected static IPv4 addresses.
3. Both containers communicated successfully with their local gateways.
4. Direct communication between containers on different bridges failed.
5. Docker's current firewall configuration used `DOCKER-FORWARD` and related chains rather than the legacy isolation chain names.

**Key Finding:** Linux routing information alone does not guarantee communication between Docker networks. Docker's network architecture and packet-filtering policies also affect whether traffic can cross bridge boundaries.

## 7. Further Investigation

- Complete and verify the multi-homed container experiment.
- Identify the exact Docker rule responsible for dropping cross-network packets.
- Examine packet counters before and after ping attempts.
- Capture dropped or forwarded traffic using tcpdump or Wireshark.
- Investigate controlled communication across Docker networks.

