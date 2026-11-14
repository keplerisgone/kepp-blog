---
title: "Docker 네트워크 실험 00 — 호스트 네트워크 기준선"
date: "2026-11-09T20:04:56+09:00"
draft: false
description: "Docker 데몬 실행 전후의 인터페이스, 라우팅, 소켓 변화를 관찰한 기록."
categories: ["study", "docker-network-lab"]
tags: ["Linux", "Docker", "Networking", "Routing"]
ShowToc: true
---

## Overview

Linux 호스트의 네트워크 구성을 조사하고, Docker 데몬 실행 전후의 네트워크 환경 변화를 관찰한다.

주요 조사 대상은 다음과 같다.

- Network Interface 및 IP Address
- Routing Table 및 Default Gateway
- Docker Bridge Network
- TCP/UDP Socket

## Background

#### 6.1 IPv4 Address와 MAC Address

IPv4 주소는 네트워크 계층에서 패킷의 출발지와 목적지를 식별하는 논리적 주소다.

MAC 주소는 Ethernet이나 Wi-Fi 등의 링크 계층에서 프레임의 송신자와 수신자를 식별하는 데 사용된다.

현재 Wi-Fi 인터페이스의 주소는 다음과 같다.

```text
IPv4: 172.30.1.8/24
MAC:  00:93:37:de:5c:51
```

IPv4 주소는 32비트이며, 일반적인 Ethernet MAC 주소는 48비트다.

#### 6.2 Routing과 ARP

`8.8.8.8`로 패킷을 전송하는 상황을 가정한다.

1. Routing Table에서 목적지에 적합한 경로를 조회한다.
2. 기본 경로를 선택하고 Next Hop을 `172.30.1.254`로 결정한다.
3. Neighbor Table에 Next Hop의 MAC 주소가 없다면 ARP를 통해 조회한다.
4. 해당 MAC 주소를 수신 대상으로 하는 링크 계층 프레임을 전송한다.

이때 IP 패킷의 목적지는 `8.8.8.8`이고, 첫 번째 링크에서의 수신 대상은 게이트웨이다.

일반적으로 NAT가 없다면 라우터를 거쳐도 IP 패킷의 최종 목적지는 유지되지만, 링크 계층의 수신 MAC 주소는 각 홉에서 변경된다.

## Environment

| 항목 | 환경 |
|---|---|
| OS | Arch Linux (Omarchy) |
| Docker | 29.9.1 |
| Docker Context | default |
| Network | Wi-Fi |
| VPN | Tailscale |

## Implementation / Experiment

### Host Network

#### 3.1 Network Interfaces

`ip -br addr`, `ip -br link`를 이용해 인터페이스를 조사했다.

| Interface | IPv4 | 역할 |
|---|---|---|
| lo | 127.0.0.1/8 | Loopback |
| enp3s0 | 없음 | Ethernet (연결되지 않음) |
| wlo1 | 172.30.1.8/24 | Wi-Fi |
| tailscale0 | 100.103.115.78/32 | VPN |

Network Interface는 Linux 커널이 네트워크를 통해 데이터를 송수신하기 위해 사용하는 논리적인 접점이다.

물리적 NIC뿐만 아니라 Loopback, VPN Tunnel, Linux Bridge 등 가상 인터페이스도 포함한다.

#### 3.2 Routing Table

```text
default via 172.30.1.254 dev wlo1 proto dhcp src 172.30.1.8 metric 600
172.30.1.0/24 dev wlo1 proto kernel scope link src 172.30.1.8 metric 600
```

Routing Table은 목적지 IP 주소를 기준으로 패킷을 전달할 경로를 결정하는 데 사용된다.

- `dev`: 패킷을 내보낼 네트워크 인터페이스
- `via`: 다음 홉(Next Hop)의 IP 주소
- `default`: 다른 경로와 일치하지 않을 때 사용하는 경로
- `/24`: 네트워크 주소의 앞 24비트를 Prefix로 사용

현재 호스트는 `172.30.1.0/24` 네트워크에 직접 연결되어 있으며, 외부 네트워크로 향하는 패킷은 기본 게이트웨이 `172.30.1.254`로 전달한다.

여러 경로가 목적지에 일치할 경우 가장 긴 Prefix를 가진 경로를 우선 선택한다(Longest Prefix Match).

### Docker Network

#### 4.1 Docker Daemon 실행 전후 비교

초기 상태에서 Docker 데몬은 비활성화되어 있었다.

```bash
systemctl is-active docker
# inactive
```

Docker CLI는 설치되어 있었지만 일반 사용자로 Docker API에 접근하려 할 때 Socket 권한 오류가 발생했다.

이후 Docker 데몬을 실행하고 네트워크 상태를 다시 조사했다.

| 항목 | 실행 전 | 실행 후 |
|---|---|---|
| docker0 | 없음 | 172.17.0.1/16 |
| Docker Route | 없음 | 172.17.0.0/16 |
| Default Route | wlo1 | wlo1 |
| Wi-Fi IP | 172.30.1.8 | 172.30.1.8 |

Docker 데몬 실행 후 `docker0` 인터페이스와 해당 서브넷 경로가 생성된 것을 확인했다.

```text
docker0 DOWN 172.17.0.1/16
```

```text
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```

`docker0`는 Linux Bridge 인터페이스이며, 컨테이너 연결이 없는 현재 상태에서는 링크가 DOWN으로 표시되었다.

#### 4.2 Docker Network Inspection

```bash
sudo docker network ls
sudo docker network inspect bridge
```

확인된 기본 네트워크:

| Network | Driver |
|---|---|
| bridge | bridge |
| host | host |
| none | null |

기본 Bridge Network의 설정은 다음과 같다.

| 항목 | 값 |
|---|---|
| Network Name | bridge |
| Driver | bridge |
| Linux Interface | docker0 |
| Subnet | 172.17.0.0/16 |
| Gateway | 172.17.0.1 |
| IPv4 | Enabled |
| IPv6 | Disabled |
| Connected Containers | 0 |

주요 옵션:

```text
enable_icc = true
enable_ip_masquerade = true
host_binding_ipv4 = 0.0.0.0
```

Docker의 `bridge`는 논리적인 네트워크 객체이며, `docker0`는 이를 구현하는 Linux Bridge 인터페이스다.

Docker가 보고하는 Subnet 및 Gateway가 Linux에서 관찰한 인터페이스 주소와 일치함을 확인했다.

IP Masquerading 옵션도 활성화되어 있었다. 실제 NAT 규칙과 패킷 주소 변환 과정은 후속 실험에서 검증한다.

### Socket Investigation

다음 명령어로 현재 호스트의 TCP/UDP 소켓을 조사했다.

```bash
sudo ss -lntup
```

주요 관찰 결과:

| Local Address | Port | Process | 용도 |
|---|---|---|---|
| 0.0.0.0 | 22/TCP | sshd | SSH |
| 127.0.0.1 | 631/TCP | cupsd | Printing |
| 127.0.0.1 | 38105/TCP | containerd | Local Service |
| 172.17.0.1 | 53/TCP, UDP | systemd-resolve | DNS |
| 127.0.0.53 | 53/TCP, UDP | systemd-resolve | DNS |
| 0.0.0.0 | 5353/UDP | avahi-daemon 등 | mDNS |
| 0.0.0.0 | 41641/UDP | tailscaled | VPN |

#### 5.1 Binding Address

- `127.0.0.1`: 동일 Network Namespace 내부에서 접근 가능한 Loopback 주소
- `0.0.0.0`: 모든 로컬 IPv4 주소에 바인딩
- `172.17.0.1`: Docker Bridge의 특정 IPv4 주소에 바인딩

실제 접근 가능 여부는 라우팅과 방화벽 등의 네트워크 정책에도 영향을 받는다.

#### 5.2 DNS Observation

`systemd-resolved`가 Docker Bridge 주소인 `172.17.0.1`의 TCP/UDP 53번 포트에서 수신 중인 것을 확인했다.

이 서비스가 Docker 네트워크에서 실제로 어떻게 사용되는지는 아직 검증하지 않았다.

## Results

Docker 데몬 실행 후 `docker0`와 `172.17.0.0/16` 경로가 추가되었다. 기존 Wi-Fi 주소와 기본 경로는 유지되었다. 상세 관찰 결과는 위 실험의 표와 명령 출력에 기록했다.

## Problems and Solutions

Docker CLI로 API에 접근할 때 소켓 권한 오류가 발생했다. 이후 네트워크 조회에는 `sudo docker network ls`와 `sudo docker network inspect bridge`를 사용했다.

## Lessons Learned

이번 실험을 통해 다음 사항을 확인했다.

1. Linux는 물리적·가상 네트워크 인터페이스를 공통된 추상화로 관리한다.
2. Routing Table은 목적지 IP 주소를 기준으로 출력 인터페이스와 Next Hop을 결정하는 데 사용된다.
3. Docker 데몬 실행 후 `docker0`와 `172.17.0.0/16` 경로가 추가되었다.
4. Docker 기본 Bridge Network의 설정은 Linux 커널에서 관찰한 인터페이스 정보와 일치했다.
5. Socket의 바인딩 주소에 따라 연결을 수신하는 로컬 주소 범위가 달라진다.

### Further Investigation

- Network Namespace가 격리하는 네트워크 자원은 무엇인가?
- veth pair는 서로 다른 Namespace를 어떻게 연결하는가?
- Linux Bridge는 Ethernet Frame을 어떻게 전달하는가?
- Docker는 IP Forwarding과 NAT를 어떻게 구성하는가?
- Docker 컨테이너가 외부 네트워크와 통신할 때 실제 패킷 경로는 어떻게 형성되는가?

이 질문들은 후속 실험에서 직접 검증한다.

## References

별도의 외부 참고 자료는 원문에 기록되어 있지 않다.

