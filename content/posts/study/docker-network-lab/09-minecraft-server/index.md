---
title: "Docker 네트워크 실험 09 — Docker Compose로 Vanilla Minecraft 서버 배포"
date: "2026-11-17T00:00:00+09:00"
draft: false
description: "Docker Compose로 Vanilla Minecraft 서버를 배포하고 포트 공개, bind mount, 클라이언트 접속을 확인한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Docker Compose", "Networking", "Minecraft", "TCP"]
aliases: ["/posts/study/docker-network-lab/08-minecraft-server/"]
ShowToc: true
---

## 1. Objective

Deploy a functional vanilla Minecraft Java Edition server in a Docker container. The server uses Mojang's vanilla implementation rather than Paper.

Apply the Docker networking concepts studied in previous experiments to a real TCP-based application.

**Success Criteria**
- Configure a Minecraft server using Docker Compose.
- Persist world data outside the container.
- Publish TCP port 25565 on the host.
- Verify the server's operational state.
- Successfully connect using Minecraft Java Edition.

## 2. Network Topology

```text
Minecraft Java Client
        |
   localhost:25565
        |
  Linux Host (Omarchy)
        |
  Docker Port Publishing
        |
    Docker Bridge
        |
      veth pair
        |
  Minecraft Container
        |
    TCP 25565
        |
  Minecraft Java Server
```

The Minecraft client and Docker host run on the same machine.

The server is published only through the host's loopback address.

## 3. Experiment

### 3.1 Prepare the Project

```bash
cd ~/project/docker-network-lab
mkdir -p minecraft
```

Project structure:

```text
docker-network-lab/
└── minecraft/
    ├── compose.yaml
    └── data/
```

The `data/` directory is used to preserve Minecraft world data and server configuration.

### 3.2 Configure Docker Compose

Create `minecraft/compose.yaml`:

```yaml
services:
  minecraft:
    image: itzg/minecraft-server:latest
    ports:
      - "127.0.0.1:25565:25565"
    environment:
      EULA: "TRUE"
      TYPE: "VANILLA"
      MEMORY: "2G"
    volumes:
      - ./data:/data
    stdin_open: true
    tty: true
```

The configuration uses the community-maintained `itzg/minecraft-server` image. For setup instructions and configuration options, see the [Minecraft Server on Docker documentation](https://docker-minecraft-server.readthedocs.io/en/latest/).

`EULA: "TRUE"` indicates acceptance of the Minecraft EULA.

**Configuration Analysis**

| Option | Purpose |
|---|---|
| `image` | Minecraft Java server image |
| `ports` | Publish TCP port 25565 |
| `EULA` | Record EULA acceptance |
| `TYPE` | Select the vanilla server |
| `MEMORY` | Configure Java heap memory |
| `volumes` | Persist world and server data |
| `stdin_open` | Keep standard input open |
| `tty` | Allocate a pseudo-terminal |

### 3.3 Understand Persistent Storage

```yaml
volumes:
  - ./data:/data
```

This is a bind mount.

The host directory `./data` is mounted as `/data` inside the container.

It allows world files and server configuration to remain available when the container is removed or recreated, provided that the host directory is preserved.

This is important because a container's writable filesystem should not be treated as permanent storage.

### 3.4 Deploy the Server

Validate the configuration:

```bash
sudo docker compose -f minecraft/compose.yaml config
```

Start the server:

```bash
sudo docker compose -f minecraft/compose.yaml up -d
```

Inspect the logs:

```bash
sudo docker compose -f minecraft/compose.yaml logs --tail=50
```

The startup logs can be used to identify initialization failures, server configuration errors, and world-generation progress.

### 3.5 Connect with Minecraft Java Edition

Launch Minecraft Java Edition and select:

```text
Multiplayer
    |
Direct Connection
    |
localhost:25565
```

**Observed result:**

Successfully connected to the Minecraft server and entered the world.

This verified that the server was functioning and accessible through Docker's published port.

### 3.6 Verify the Server Status

```bash
sudo docker compose -f minecraft/compose.yaml ps
```

**Observed result:**

```text
NAME                    STATUS
minecraft-minecraft-1   Up 11 minutes (healthy)
```

The image was:

```text
itzg/minecraft-server:latest
```

The `healthy` state indicates that the container's configured health check was passing.

### 3.7 Verify Port Publishing

```bash
sudo docker compose -f minecraft/compose.yaml port minecraft 25565
```

**Observed result:**

```text
127.0.0.1:25565
```

The container was configured with:

```text
127.0.0.1:25565 -> 25565/tcp
```

This confirmed that Minecraft's TCP service was published through the host's loopback interface.

## 4. Analysis

### Connection and Gameplay Results

The first screenshot shows the client configured to connect to `localhost:25565`, matching the loopback address and port published by Docker Compose.

![Minecraft Direct Connection screen with localhost:25565](image%20%281%29.png)

*Client connection settings for the Docker-hosted server.*

The second screenshot shows the client inside the world after connecting. Together with the observed server status and port mapping, this supports the successful end-to-end connection reported in this experiment: the client reached the containerized server through the host's published TCP port.

![Minecraft gameplay after joining the Docker-hosted vanilla server](image%20%282%29.png)

*The world loaded successfully after joining the server.*

These results demonstrate a working local connection. They do not establish access from another machine, since the published address is bound to `127.0.0.1`. Packet captures and a container-recreation test would provide additional evidence for the network path and world persistence, respectively.

### Docker Compose vs. docker run

In previous experiments, containers were created using individual `docker run` commands.

Docker Compose allows container configuration to be declared in a YAML file.

This provides a reproducible definition of the container's image, published ports, environment variables, and persistent storage.

### TCP Port Publishing

Minecraft Java Edition uses TCP port 25565 by default.

The server listens on port 25565 inside the container.

Docker exposes that service through port 25565 on the Linux host.

The client connects using:

```text
localhost:25565
```

The request reaches the Minecraft server through Docker's configured network forwarding mechanism.

### Network Isolation

The Minecraft server runs inside a container with its own networking environment.

Docker provides connectivity through its network driver and published port configuration.

The server does not require direct access to the host's physical network interface.

### Persistent World Data

World data is stored using a host bind mount.

This separates the container's runtime lifecycle from persistent game data.

The configuration can therefore be reproduced without intentionally discarding the existing Minecraft world.

## 5. Personal Reflection

Seeing how little configuration was needed to start a Minecraft server with Docker made me reconsider my earlier [Minecraft Server Generator project](https://github.com/keplerisgone/minecraft-server). I had built a tool to simplify server setup, yet an existing container image already handled much of that work.

Building the generator was still a useful learning experience. This experiment reminded me to research existing tools before starting a new project, understand which problems they already solve, and identify where my own implementation could add value. For future projects, I want to make that comparison part of the initial design process.

## 6. Conclusion

Successfully deployed a functional Minecraft Java Edition server using Docker Compose.

**Verified Findings**
1. Docker Compose successfully deployed the Minecraft server.
2. The container passed its configured health check.
3. TCP port 25565 was published on `127.0.0.1`.
4. Minecraft Java Edition successfully connected to the server.
5. Persistent world storage was configured using a bind mount.

**Key Finding:** Docker Compose can simplify the deployment of stateful TCP services by managing container configuration, network publishing, and persistent storage declaratively.

## 7. Further Investigation

- Capture the Minecraft TCP three-way handshake.
- Observe packets through the Docker Bridge.
- Compare host-side and container-side TCP connections.
- Analyze packet sizes and traffic patterns during gameplay.
- Reproduce network disconnections and investigate recovery behavior.

These topics will be explored in the following experiments.

