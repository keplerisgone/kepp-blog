---
title: "Docker 네트워크 실험 10 — Wireshark로 Minecraft TCP 트래픽 분석"
date: "2026-11-19T00:00:00+09:00"
draft: false
description: "Minecraft TCP 핸드셰이크와 바이트 스트림을 분석하고 loopback과 Docker Compose bridge의 패킷 관측 결과를 비교한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Networking", "Minecraft", "TCP", "Wireshark"]
ShowToc: true
---

## 1. Objective

Analyze TCP communication between a Minecraft Java client and a Docker-hosted Minecraft server.

Compare packet captures from the host loopback interface and the Docker Compose bridge to understand the network path used by published container ports.

**Success Criteria**
- Capture Minecraft TCP traffic using Wireshark.
- Identify the TCP three-way handshake.
- Understand TCP sequence numbers, acknowledgments, and flags.
- Compare host-side and Docker-side traffic.
- Identify differences in IP addresses and TCP connections.

## 2. Network Topology

The Minecraft server was deployed in Experiment 09 using Docker Compose.

```text
Minecraft Java Client
127.0.0.1
       |
       | TCP connection
       v
127.0.0.1:25565
       |
   Docker Port Publishing
       |
       v
Compose Bridge
br-edfe3dec3731
172.19.0.1
       |
       | veth pair
       v
Minecraft Container
172.19.0.2:25565
```

Two observation points were used:

1. `lo` — Host loopback traffic.
2. `br-edfe3dec3731` — Docker Compose bridge traffic.

## 3. Experiment

### 3.1 Configure Wireshark

![Wireshark welcome screen without local capture interfaces](image%20%281%29.png)

*Wireshark initially listed remote capture sources but no local interfaces.*
Initially, Wireshark did not display local network interfaces.

Running `dumpcap -D` failed:

```text
bash: /usr/bin/dumpcap: Permission denied
```

The executable permissions were:

```text
-rwxr-x--- root wireshark /usr/bin/dumpcap
```

The current user was not a member of the `wireshark` group.

The issue was resolved by adding the user to the group:

```bash
sudo usermod -aG wireshark "$USER"
newgrp wireshark
```

Verification:

```bash
id -nG
dumpcap -D
```

The command subsequently listed interfaces including `lo`, `docker0`, and the Docker Compose bridge.

**Finding:** Packet capture permissions depend on executable permissions and Linux security configuration. Wireshark's GUI can analyze saved captures without live-capture permissions.

### 3.2 Capture Minecraft Traffic on Loopback

Selected interface:

```text
lo
```

Wireshark display filter:

```text
tcp.port == 25565
```

Minecraft was accessed through:

```text
localhost:25565
```

**Observed connection:**

```text
127.0.0.1:42816 → 127.0.0.1:25565
```

The capture contained TCP data packets, acknowledgments, and connection termination packets.

![Minecraft loopback TCP packets filtered by port 25565](image%20%282%29.png)

*Loopback traffic includes PSH-ACK data segments and FIN-ACK connection termination packets.*

### 3.3 Analyze TCP Three-Way Handshake

Applied filter:

```text
tcp.port == 25565 && tcp.flags.syn == 1
```

Four SYN-related packets were observed, corresponding to two connection attempts.

![Wireshark SYN filter showing two TCP connection attempts](image%20%284%29.png)

*The SYN filter selects SYN and SYN-ACK packets; the final ACK of each handshake is excluded by this filter.*

| Packet | Direction | Flags |
|---|---|---|
| 5 | 40784 → 25565 | SYN |
| 6 | 25565 → 40784 | SYN, ACK |
| 33 | 42816 → 25565 | SYN |
| 34 | 25565 → 42816 | SYN, ACK |

The full handshake of TCP stream 1 was then inspected.

```text
No. 33  Client → Server  SYN       Seq=0
No. 34  Server → Client  SYN, ACK  Seq=0 Ack=1
No. 35  Client → Server  ACK       Seq=1 Ack=1
```

![TCP stream 1 showing the complete three-way handshake](image%20%285%29.png)

*Packets 33–35 show SYN, SYN-ACK, and ACK, followed by application data and acknowledgments.*

**Analysis**

TCP establishes connections through three steps:

1. The client sends SYN to initiate a connection.
2. The server responds with SYN-ACK.
3. The client acknowledges the response with ACK.

Wireshark displays relative TCP sequence numbers by default.

The SYN flag is primarily used during connection establishment, explaining why only four SYN-related packets appeared among thousands of captured packets.

### 3.4 Analyze TCP Flags

The capture contained many packets marked:

```text
[ACK]
[PSH, ACK]
[FIN, ACK]
```

| Flag | Purpose |
|---|---|
| SYN | Initiate TCP connection establishment |
| ACK | Acknowledge received bytes |
| PSH | Request prompt delivery of available data to the receiving application |
| FIN | Indicate the end of one direction of data transmission |

An ACK number represents the next byte expected by the receiver.

For example:

```text
Seq=1
TCP Len=17
```

After receiving these bytes in order, the receiver can acknowledge them using:

```text
Ack=18
```

TCP acknowledges byte positions rather than individual packets.

The PSH flag does not indicate elevated network priority.

### 3.5 Inspect TCP Stream Data

Wireshark's **Follow TCP Stream** feature was used to inspect the connection.

The selected conversation reported approximately:

```text
Client packets: 724
Server packets: 715
Conversation data: 2,647 kB
```

Much of the application payload appeared unreadable when displayed as ASCII.

![Follow TCP Stream view of Minecraft binary payload](image%20%283%29.png)

*The ASCII stream view reports 724 client packets, 715 server packets, and approximately 2,647 kB of conversation data.*

**Analysis**

Minecraft Java Edition uses a binary application protocol.

The captured payload may include structured binary data, compressed content, or encrypted application data.

TCP transports an ordered byte stream without interpreting Minecraft's gameplay messages.

Therefore, unreadable ASCII output does not imply packet corruption.

### 3.6 Locate the Minecraft Docker Bridge

Initially, capturing on `docker0` did not reveal Minecraft traffic.

The container's network configuration was inspected:

```bash
sudo docker inspect minecraft-minecraft-1 \
  --format '{{json .NetworkSettings.Networks}}'

sudo docker network ls
bridge link show
```

**Observed configuration:**

```text
Docker network: minecraft_default
Linux Bridge:   br-edfe3dec3731
Gateway:        172.19.0.1
Container IP:   172.19.0.2
```

The Minecraft container was connected to its dedicated Compose network rather than Docker's default `docker0` bridge.

**Finding:** Docker Compose normally creates a separate user-defined network for a project unless another network configuration is specified.

### 3.7 Capture Docker-Side TCP Traffic

Selected Wireshark interface:

```text
br-edfe3dec3731
```

Display filter:

```text
tcp.port == 25565
```

**Observed traffic:**

```text
172.19.0.1:57078 → 172.19.0.2:25565
172.19.0.2:25565 → 172.19.0.1:57078
```

The Docker-side traffic used different socket endpoints from the loopback connection.

| Property | Loopback capture | Docker bridge capture |
|---|---|---|
| Interface | lo | br-edfe3dec3731 |
| Source IP | 127.0.0.1 | 172.19.0.1 |
| Destination IP | 127.0.0.1 | 172.19.0.2 |
| Observed client-side port | 42816 | 57078 |
| Server port | 25565 | 25565 |

These observations were obtained from separate captures, so the specific connections were not conclusively correlated packet by packet.

### 3.8 Interpret Docker Port Publishing

Experiment 08 previously confirmed that Docker configured a userland proxy for a published HTTP port.

The Minecraft captures are consistent with a proxy-based path:

```text
Minecraft Client
       |
       | TCP Connection A
       v
Docker Published Port
       |
       | Proxy / Forwarding
       v
Docker Bridge
       |
       | TCP Connection B
       v
Minecraft Container
```

In a userland-proxy implementation, the proxy accepts one TCP connection and establishes another toward the container.

The two connections have independent TCP state, including source ports and sequence numbers.

The captures support this interpretation, although direct inspection of the Minecraft `docker-proxy` process and simultaneous correlation of both connections were not performed.

### 3.9 Observe Large TCP Segments

The Docker bridge capture contained packets with reported lengths exceeding ordinary Ethernet MTU sizes, including frames around 65 KB.

Such observations can result from Linux networking offload and aggregation mechanisms, including GSO and GRO.

A large frame visible in a virtual-interface capture does not necessarily correspond to one physical Ethernet frame transmitted on the wire.

## 4. Key Findings

- TCP connection establishment was verified through SYN, SYN-ACK, and ACK packets.
- TCP sequence and acknowledgment numbers track byte positions.
- Application data was carried using TCP segments, including PSH-ACK segments.
- Minecraft's binary payload was not directly readable as ordinary ASCII.
- Docker Compose used a dedicated Linux Bridge rather than `docker0`.
- Loopback and Docker-side captures showed different socket endpoints.
- TCP connection establishment succeeded independently of the Minecraft authentication failure.

## 5. Conclusion

Successfully captured and analyzed Minecraft TCP traffic using Wireshark.

The experiment demonstrated TCP connection establishment, reliable byte-stream communication, and differences between packet observations at two Docker networking interfaces.

**Key Finding:** Observing traffic at multiple network interfaces reveals different stages of a container's communication path. Docker port publishing can introduce additional forwarding or proxy behavior beyond ordinary container-to-container switching.

## 6. Further Investigation

- Correlate simultaneous loopback and bridge captures.
- Inspect established `docker-proxy` sockets.
- Analyze TCP retransmissions and round-trip time.
- Capture a complete successful Minecraft gameplay session.
- Compare packet lengths with network offload features enabled and disabled.

