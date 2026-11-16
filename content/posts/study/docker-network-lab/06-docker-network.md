---
title: "Docker 네트워크 실험 06 — Docker Bridge와 컨테이너 네트워크"
date: "2026-11-16T00:00:00+09:00"
draft: false
description: "Docker 기본 bridge에서 컨테이너 통신을 확인하고 veth, docker0, MAC 주소 학습을 살펴본 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Networking", "Network Namespace", "veth", "Linux Bridge"]
ShowToc: true
---

## 1. Objective

Investigate how Docker automatically constructs container networks using Linux kernel networking primitives.

Compare Docker's default bridge network with the manually configured Network Namespaces, veth pairs, and Linux Bridge from previous experiments.

**Success Criteria**
- Create two containers on Docker's default bridge.
- Identify Docker-created veth interfaces.
- Verify the connection between veth interfaces and `docker0`.
- Establish container-to-container communication.
- Inspect MAC address learning through the bridge FDB.

## 2. Network Topology

```text
                  Linux Host
                      |
                  docker0
                172.17.0.1/16
                 /         \
                /           \
        veth6b5573e     vethde4c97d
              |               |
           veth pair       veth pair
              |               |
             eth0            eth0
              |               |
          netlab-a         netlab-b
        172.17.0.2       172.17.0.3
              |               |
       Network Namespace A / B
```

Both containers are connected to the same Linux Bridge, `docker0`, through separate veth pairs.

## 3. Experiment

### 3.1 Inspect Docker Environment

```bash
systemctl is-active docker
sudo docker ps -a
sudo docker images
sudo docker network ls
ip -br link
```

**Observations**
- Docker daemon: `active`
- Existing containers: None
- Existing images: None
- Docker networks: `bridge`, `host`, `none`
- `docker0`: Initially DOWN
- `br-lab`: Still UP from the previous experiment

### 3.2 Create the First Container

```bash
sudo docker pull alpine:3.22

sudo docker run -d \
  --name netlab-a \
  --network bridge \
  alpine:3.22 sleep 3600
```

Inspect the container:

```bash
sudo docker inspect -f \
  '{{range .NetworkSettings.Networks}}IP={{.IPAddress}} Gateway={{.Gateway}}{{end}}' \
  netlab-a

sudo docker exec netlab-a ip addr
sudo docker exec netlab-a ip route
ip -br link
```

**Observations**

| Property | Value |
|---|---|
| Container | netlab-a |
| IPv4 | 172.17.0.2/16 |
| Gateway | 172.17.0.1 |
| Container interface | eth0 |
| Host-side veth | veth6b5573e |
| Bridge | docker0 |

The host's `docker0` interface changed from DOWN to UP.

A new host-side veth interface appeared after creating the container.

Inside the container, the routing table contained:

```text
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link src 172.17.0.2
```

**Analysis**

Docker automatically configured a network interface, an IP address, and routing information inside the container.

The newly created host-side veth interface provided connectivity between the container's isolated networking environment and the host's Docker Bridge.

### 3.3 Verify the Bridge Connection

```bash
bridge link show
sudo docker network inspect bridge
```

**Observation**

```text
veth6b5573e ... master docker0 state forwarding
```

Docker's network inspection also showed that `netlab-a` was connected to the default bridge.

The network configuration contained:

```text
Subnet:  172.17.0.0/16
Gateway: 172.17.0.1
```

**Analysis**

The `master docker0` field confirms that the host-side veth interface is attached as a bridge port.

This is equivalent to manually attaching a veth endpoint to `br-lab` using:

```bash
ip link set <interface> master br-lab
```

### 3.4 Create a Second Container

```bash
sudo docker run -d \
  --name netlab-b \
  --network bridge \
  alpine:3.22 sleep 3600
```

Inspect:

```bash
sudo docker network inspect bridge
ip -br link
bridge link show
```

**Observations**

| Container | IPv4 | Host-side veth |
|---|---|---|
| netlab-a | 172.17.0.2 | veth6b5573e |
| netlab-b | 172.17.0.3 | vethde4c97d |

Both veth interfaces were connected to `docker0` and entered the forwarding state.

### 3.5 Verify Container Connectivity

```bash
sudo docker exec netlab-a ping -c 3 172.17.0.3
```

**Result**

```text
3 packets transmitted
3 packets received
0% packet loss
```

The containers successfully communicated through Docker's default bridge.

### 3.6 Inspect the Forwarding Database

```bash
bridge fdb show br docker0 | grep -v permanent
```

**Observed output**

```text
42:d2:ce:a2:a7:15 dev veth6b5573e master docker0
46:48:9b:f2:a3:29 dev vethde4c97d master docker0
```

The bridge learned which MAC addresses were reachable through each veth port.

This confirms the same Layer 2 MAC learning mechanism observed in Experiment 03.

## 4. Analysis

### Docker vs. Manual Linux Networking

| Manual configuration | Docker configuration |
|---|---|
| Network Namespace | Container Network Namespace |
| Manually created veth pair | Docker-created veth pair |
| br-lab | docker0 |
| Manual IP configuration | Docker IPAM |
| Manual route configuration | Docker-managed routes |
| Linux Bridge FDB | docker0 FDB |

Docker uses existing Linux kernel networking features to provide container network isolation and connectivity.

Each container has an isolated networking environment, while veth pairs allow controlled communication with the host's Linux Bridge.

### Packet Forwarding

For communication between `netlab-a` and `netlab-b`:

1. `netlab-a` selects the directly connected route to `172.17.0.3`.
2. Neighbor resolution determines the destination MAC address if necessary.
3. An Ethernet frame leaves the container through `eth0`.
4. Its veth peer delivers the frame to the host-side bridge port.
5. `docker0` forwards the frame according to its FDB.
6. The frame reaches `netlab-b` through its corresponding veth pair.

This communication does not require Layer 3 routing between different IP subnets because both containers belong to `172.17.0.0/16`.

## 5. Conclusion

Successfully reproduced and verified Docker's default bridge networking topology.

**Verified Findings**
1. Docker creates a veth pair for a container connected to its default bridge.
2. The container receives its own interface, IP address, and routing configuration.
3. Host-side veth interfaces are attached to `docker0`.
4. Two containers on the same bridge can communicate using IPv4 addresses.
5. The Linux Bridge dynamically learns MAC-to-port mappings.

**Key Finding:** Docker Bridge Networking is built upon the same Linux networking primitives previously configured manually: Network Namespaces, veth pairs, Linux Bridges, and routing tables.

## 6. Further Investigation

- Inspect the container's network namespace directly from the host.
- Examine Docker-managed IP forwarding and NAT rules.
- Compare default bridge networking with user-defined networks.
- Trace container traffic using tcpdump.

