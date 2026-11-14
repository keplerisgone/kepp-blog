---
title: "Day 15: TCP Socket과 Echo"
date: "2026-09-01T14:25:14+09:00"
lastmod: "2026-09-01T14:25:14+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-15-tcp-socket과-echo/"
weight: 15
ShowToc: true
---

Loopback 주소 `127.0.0.1:9090`에서 동작하는 blocking TCP echo server와 client를 구현했다. `nc`와 Bash `/dev/tcp`로 연결을 먼저 관찰한 뒤 C socket API로 같은 흐름을 구성했다.

## TCP와 endpoint

IP address는 network에서 host 또는 interface를 식별하고, port는 host 안에서 communication endpoint를 구분한다. 이번 실습은 외부 interface가 아닌 현재 host 자신을 가리키는 loopback 주소를 사용했다.

```text
Server endpoint: 127.0.0.1:9090
Protocol: TCP
```

TCP는 연결 지향적이고 전송 순서를 유지하는 byte stream이다. Application message boundary는 보존하지 않으므로 한 번의 `write()`와 한 번의 `read()`가 일대일로 대응한다고 가정할 수 없다.

## Server socket lifecycle

Server는 다음 순서로 구성했다.

```text
socket()
→ bind(127.0.0.1:9090)
→ listen()
→ accept()
→ read()/write()
→ close()
```

`socket(AF_INET, SOCK_STREAM, 0)`은 IPv4 TCP socket을 만들고 process에서 사용할 FD를 반환한다. `inet_pton()`으로 문자열 IPv4 주소를 `struct in_addr` 형식으로 변환하고, `htons()`로 port를 network byte order로 변환했다.

`bind()`는 socket에 local address를 지정하고, `listen()`은 socket을 연결 요청을 기다리는 상태로 전환한다. `ss -ltnp`에서 `127.0.0.1:9090`의 `LISTEN` 상태를 확인했다.

## Listening FD와 connected FD

`accept()`는 listening socket에 도착한 연결 하나를 받아 새로운 connected FD를 반환한다.

```text
server_fd
→ 새로운 연결 요청을 받는 listening FD

client_fd
→ 특정 client와 데이터를 주고받는 connected FD
```

Listening FD와 connected FD는 역할과 lifetime이 다르다. 이번 server는 client 하나만 처리한 뒤 `client_fd`, `server_fd` 순서로 정리한다.

## Read loop와 EOF

Connected FD에서 `read()`를 반복하며 다음 반환값을 구분했다.

```text
n > 0   → n바이트 수신
n == 0  → peer가 송신 방향을 닫아 EOF 도달
n == -1 → 오류
```

TCP는 byte stream이므로 한 번의 `read()`가 client가 보낸 전체 데이터를 반환한다고 보장하지 않는다. Buffer가 C string처럼 NUL-terminated 된다고 가정하지 않고 실제 길이 `n`을 기준으로 데이터를 처리했다.

## Partial write

한 번의 `write()`도 요청한 전체 길이를 모두 처리한다고 보장하지 않는다. 지금까지 전송한 길이 `sent`를 누적하면서 남은 영역을 반복해 전송했다.

```text
전송 시작 위치 = buffer + sent
남은 길이      = n - sent
```

`write()`가 양수를 반환하면 그만큼 `sent`를 증가시켰다. `errno == EINTR`이면 같은 위치부터 재시도하고, 다른 오류나 전송 진척이 없는 경우에는 cleanup으로 이동했다.

## Client와 half-close

Client는 다음 순서로 동작한다.

```text
socket()
→ connect(127.0.0.1:9090)
→ argv message 전송
→ shutdown(SHUT_WR)
→ echo를 EOF까지 수신
→ close()
```

`connect()`는 원격 server endpoint와 TCP 연결을 만든다. 메시지를 모두 보낸 뒤 `shutdown(socket_fd, SHUT_WR)`을 호출해 송신 방향만 닫았다.

```text
Client → Server 방향: EOF 전달
Server → Client 방향: 계속 열려 있어 echo 수신 가능
```

Server는 EOF를 확인하고 echo 전송을 마친 뒤 connected FD를 닫는다. Client는 그 결과로 수신 방향의 EOF까지 확인할 수 있다.

## Error handling과 cleanup

각 socket API의 반환값을 검사하고, 열린 자원의 수에 따라 공통 cleanup label로 이동했다.

```text
cleanup_client
→ client_fd close
→ cleanup_server로 이어짐
→ server_fd close
```

`perror()`는 원래 오류가 발생한 지점에서 먼저 호출했다. Cleanup 중 `close()`가 `errno`를 변경하기 전에 실제 실패 원인을 기록하기 위해서다. Cleanup label로만 아래 방향으로 이동하는 제한적인 `goto`가 C system code의 중복 정리에 유용하다는 점도 확인했다.

## 관찰

연결 상태는 다음 명령으로 확인했다.

```bash
ss -ltnp | rg ':9090'
ss -tnp | rg ':9090'
```

`LISTEN`과 `ESTABLISHED` 상태를 관찰했고, `strace`로 client의 system call 흐름을 확인했다.

```bash
strace -yy \
  -e trace=socket,bind,listen,accept,connect,read,write,shutdown,close \
  ./network/tcp-echo-client "hello echo"
```

```text
socket → connect → write → shutdown → read → EOF → close
```

## Linux System Software 직무와의 연결

`sysmon-agent`에서는 collector가 만든 metric을 network thread가 TCP로 전달할 예정이다. 이번 실습에서 익힌 socket lifetime, byte-stream 처리, partial write, EOF와 cleanup은 해당 network path의 기반이 된다.

## Learned

- TCP는 message가 아니라 순서가 유지되는 byte stream이다.
- Server의 listening FD와 client별 connected FD는 역할이 다르다.
- `accept()`는 특정 client와 연결된 새로운 FD를 반환한다.
- `read() == 0`은 peer의 송신 방향에서 EOF가 도착했다는 뜻이다.
- `read()`와 `write()`는 application message 단위와 일치하지 않을 수 있다.
- Partial write는 offset과 남은 길이를 갱신하며 처리한다.
- `shutdown(SHUT_WR)`로 수신 방향을 유지한 채 송신 완료를 알릴 수 있다.
- 공통 cleanup label로 여러 FD의 lifetime을 관리할 수 있다.

## APIs

- `socket()`
- `inet_pton()`
- `htons()`
- `bind()`
- `listen()`
- `accept()`
- `connect()`
- `read()`
- `write()`
- `shutdown()`
- `close()`

## Commands

- `ss -ltnp`
- `ss -tnp`
- `strace -yy -e trace=...`
- Bash `/dev/tcp/127.0.0.1/9090`

## Mistakes

- 처음에는 `inet_pton()` 결과를 임시 `int`에 저장하려 했다. Destination은 `addr.sin_addr`여야 한다.
- 오류 분기마다 `close()`와 `return`을 반복해 cleanup 흐름이 복잡해졌다.
- TCP 데이터를 NUL-terminated C string처럼 취급하기 쉬웠다.

## TODO

- `read()`의 `EINTR` 재시도 처리
- Peer가 먼저 연결을 닫을 때 발생할 수 있는 `SIGPIPE` 처리
- 여러 client를 순차 또는 동시 처리하는 server 구조는 이후 학습

## Next

- `/proc/stat`을 읽고 두 시점의 CPU tick 차이로 CPU usage를 계산한다.
