---
title: "Day 8: Signal과 안전한 종료"
date: "2026-08-13T22:03:28+09:00"
lastmod: "2026-08-13T22:03:28+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-8-signal과-안전한-종료/"
weight: 8
ShowToc: true
---

이번 실습의 목표는 Linux signal의 기본 구조를 이해하고, `sigaction()`으로 `SIGINT`와 `SIGTERM` handler를 등록하여 프로그램이 안전하게 종료 요청을 처리하도록 만드는 것이다.

```text
signal 도착
→ handler가 종료 flag 변경
→ main이 flag 확인
→ main에서 출력과 자원 정리
→ 정상 종료
```

## Signal

Signal은 kernel 또는 다른 프로세스가 프로세스에 사건을 비동기적으로 알리는 방법이다.

```text
Ctrl-C           → terminal이 SIGINT 전달
kill -TERM PID   → 다른 프로세스가 SIGTERM 전달
잘못된 메모리 접근 → kernel이 SIGSEGV 전달
자식 상태 변화     → kernel이 SIGCHLD 전달
```

비동기적이라는 말은 프로그램이 특정 코드까지 도달하기를 기다리지 않고, 실행 중 어느 시점에나 signal이 전달될 수 있다는 뜻이다.

`SIGINT`와 `SIGTERM`의 기본 동작은 프로세스 종료다. `SIGINT`는 일반적으로 terminal에서 `Ctrl-C`를 눌렀을 때 foreground process group에 전달되고, `SIGTERM`은 프로세스에 정상적인 종료 기회를 주기 위한 요청으로 흔히 사용된다.

## Signal disposition과 handler

프로세스가 signal을 받았을 때 취할 동작을 signal disposition이라고 한다.

```text
기본 동작 수행
signal 무시
사용자가 등록한 handler 실행
```

Signal handler는 signal이 전달됐을 때 잠시 실행되는 함수다.

```c
void custom_handler(int signo)
```

`signo`에는 전달된 signal 번호가 들어온다. 하나의 handler를 여러 signal에 등록해도 이 번호로 어떤 signal이 전달됐는지 구분할 수 있다.

```text
SIGINT  ─┐
         ├─→ custom_handler(signo)
SIGTERM ─┘
```

Handler 함수만 정의한다고 kernel이 자동으로 실행하지는 않는다. `sigaction()`을 호출해 어떤 signal과 어떤 handler를 연결할지 kernel에 등록해야 한다.

## SIGKILL과 SIGSTOP

`SIGKILL`과 `SIGSTOP`은 handler를 등록하거나 무시할 수 없다.

```text
SIGKILL → 반드시 프로세스 종료
SIGSTOP → 반드시 프로세스 정지
```

프로세스가 오작동하더라도 kernel과 관리자가 확실하게 종료하거나 정지할 수단을 보장하기 위해서다.

## struct sigaction

`struct sigaction`은 signal 동작을 등록하기 위한 설정서 역할을 한다.

```text
struct sigaction
├─ sa_handler : 실행할 handler
├─ sa_mask    : handler 실행 중 추가로 차단할 signal 집합
└─ sa_flags   : 추가 동작 옵션
```

구조체를 0으로 초기화한 뒤 오늘 필요한 필드를 설정했다.

```c
struct sigaction action = {0};
action.sa_handler = custom_handler;
action.sa_flags = 0;
```

`custom_handler()`가 아니라 `custom_handler`를 대입했다. 함수를 지금 호출하는 것이 아니라 나중에 kernel이 호출할 함수의 주소를 전달하기 때문이다.

`sa_mask`는 `sigset_t`이므로 정수 0을 직접 대입하지 않고 전용 API로 빈 signal 집합을 만들었다.

```c
if (sigemptyset(&action.sa_mask) == -1) {
	perror("sigemptyset");
	return 1;
}
```

이번 설정에서는 handler 실행 중 추가로 차단할 signal이 없다. 다만 현재 처리 중인 동일 signal은 기본적으로 handler 실행 중 자동 차단된다.

## sigaction() 등록

`sigaction()`의 인자는 다음 의미를 가진다.

```text
sigaction(signal 번호, 새로운 설정의 주소, 이전 설정을 저장할 주소)
```

```c
sigaction(SIGINT, &action, NULL);
```

이는 다음 문장으로 해석할 수 있다.

> SIGINT의 동작을 `action`의 내용으로 변경하고, 이전 설정은 저장하지 않는다.

동일한 `action`을 `SIGTERM`에도 등록했다. 두 호출 모두 실패 시 `-1`을 반환하고 `errno`를 설정하므로 각각 검사했다.

## Handler에서 flag만 변경하기

Signal은 main이 어떤 함수를 실행하는 중에도 도착할 수 있다. 예를 들어 main의 `printf()`가 내부 buffer나 lock을 처리하는 중 handler가 다시 `printf()`를 호출하면 미완성 상태를 재사용해 상태 손상이나 deadlock이 발생할 수 있다.

이런 비동기적인 재진입 상황에서도 안전하다고 보장되는 성질을 async-signal-safe라고 한다. `printf()`와 대부분의 stdio 함수는 async-signal-safe하지 않다.

따라서 handler에서는 signal 번호를 flag에 저장하는 최소 작업만 수행했다.

```c
volatile sig_atomic_t stop_requested = 0;

void custom_handler(int signo) {
	stop_requested = signo;
}
```

실제 출력과 종료 처리는 main 실행 흐름에서 수행했다.

```text
handler 안
→ flag 변경
→ 즉시 return

main 안
→ flag 확인
→ 메시지 출력
→ 향후 자원 정리
→ 정상 종료
```

## volatile sig_atomic_t

`sig_atomic_t`는 signal handler와 일반 실행 흐름 사이에서 단순 읽기와 쓰기를 중간 상태 없이 수행할 수 있도록 C가 제공하는 정수 타입이다.

`volatile`은 이 값이 현재 보이는 실행 흐름 외부에서 비동기적으로 바뀔 수 있으므로 compiler가 이전 값을 계속 재사용하지 말고 다시 관찰하도록 한다.

```text
sig_atomic_t → handler와 main 사이의 단순 읽기·쓰기
volatile     → 비동기적인 값 변경을 compiler에 알림
```

이는 pthread 사이의 동기화 수단이 아니다. 여러 thread가 공유 데이터를 다룰 때는 mutex 또는 atomic을 사용해야 한다.

## signal-demo 구현

최종 코드는 다음과 같다.

```c
#include <stdint.h>
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

volatile sig_atomic_t stop_requested = 0;

void custom_handler(int signo) {
	stop_requested = signo;
}

int main(void) {
	struct sigaction action = {0};
	action.sa_handler = custom_handler;
	if (sigemptyset(&action.sa_mask) == -1) {
		perror("sigemptyset");
		return 1;
	}
	action.sa_flags = 0;

	if (sigaction(SIGINT, &action, NULL) == -1) {
		perror("sigaction - SIGINT");
		return 1;
	}
	if (sigaction(SIGTERM, &action, NULL) == -1) {
		perror("sigaction - SIGTERM");
		return 1;
	}

	pid_t pid = getpid();
	printf("PID: %jd\n", (intmax_t) pid);

	while (stop_requested == 0) {
		sleep(1);
	}

	printf("received signal %d, shutting down...\n", (int)stop_requested);

	return 0;
}
```

전체 흐름은 다음과 같다.

```text
sigaction 구조체 준비
→ SIGINT handler 등록
→ SIGTERM handler 등록
→ PID 출력
→ flag가 0인 동안 sleep
→ signal이 sleep을 중단
→ handler가 flag에 signal 번호 저장
→ main이 반복문을 빠져나옴
→ main에서 메시지 출력
→ 0으로 정상 종료
```

## SIGINT와 SIGTERM 테스트

Foreground에서 실행한 뒤 `Ctrl-C`를 눌렀다.

```bash
./signal-demo
```

Handler는 `SIGINT` 번호 2를 flag에 저장했고 main이 종료 메시지를 출력했다.

Background에서는 다음처럼 `SIGTERM`을 전달했다.

```bash
./signal-demo &
demo_pid=$!
sleep 1
kill -TERM "$demo_pid"
wait "$demo_pid"
echo $?
```

`SIGTERM` 번호 15를 출력한 뒤 main이 0을 반환했다.

## Handler 등록 전의 race condition

처음에는 프로그램을 background로 실행하자마자 `SIGTERM`을 보냈고 종료 상태 143이 나타났다.

```text
143 = 128 + SIGTERM(15)
```

원인은 외부 signal 전송이 `sigaction()` 등록보다 먼저 일어난 것이었다.

```text
프로세스 시작
├─ 프로그램: handler 등록 준비 중
└─ shell: 즉시 SIGTERM 전달
                ↓
       아직 handler가 없음
                ↓
       SIGTERM 기본 동작으로 종료
```

Handler는 프로그램 시작과 동시에 존재하는 것이 아니라 `sigaction()` 호출이 성공한 시점부터 적용된다. 테스트에서는 잠시 기다린 뒤 signal을 보내 해결했다.

`sleep(1)`은 관찰용으로는 충분하지만 실제 프로세스 간 준비 완료 동기화 방법으로는 부정확하다. 이후 IPC 학습에서 pipe나 다른 handshake 방법을 다룬다.

## SIGKILL 관찰

`SIGKILL`을 보내자 handler와 main의 종료 메시지가 실행되지 않고 프로세스가 즉시 종료됐다.

```text
137 = 128 + SIGKILL(9)
```

```text
SIGTERM
→ handler 실행 가능
→ main에서 정리 가능
→ 정상 종료 0

SIGKILL
→ handler 등록 불가능
→ kernel이 즉시 종료
→ 정리 코드 실행 불가능
→ 종료 상태 137
```

장시간 실행되는 daemon은 보통 `SIGTERM`을 받아 socket, file descriptor, thread 등의 자원을 정리한 뒤 종료한다. `SIGKILL`은 프로그램이 응답하지 않을 때 사용하는 최후의 수단이다.

## strace 관찰

다음 명령으로 handler 등록과 signal이 sleep을 중단하는 과정을 관찰했다.

```bash
strace \
    -e trace=rt_sigaction,clock_nanosleep \
    -e signal=SIGINT,SIGTERM \
    ./signal-demo
```

주요 결과는 다음과 같았다.

```text
rt_sigaction(SIGINT, {sa_handler=..., sa_mask=[], sa_flags=SA_RESTORER, ...}, NULL, 8) = 0
rt_sigaction(SIGTERM, {sa_handler=..., sa_mask=[], sa_flags=SA_RESTORER, ...}, NULL, 8) = 0
clock_nanosleep(CLOCK_REALTIME, 0, {tv_sec=1, ...}, ...) = 0
clock_nanosleep(..., {tv_sec=0, tv_nsec=345712292})
    = ? ERESTART_RESTARTBLOCK (Interrupted by signal)
received signal 2, shutting down...
```

두 `rt_sigaction()`에 표시된 handler 주소가 같았으므로 `SIGINT`와 `SIGTERM`이 같은 함수에 연결된 것을 확인했다. `sa_mask=[]`는 추가 차단 signal 집합이 비어 있음을 나타낸다.

코드에서는 `sa_flags = 0`으로 설정했지만 `strace`에는 `SA_RESTORER`가 보였다. 이는 handler 실행 후 원래 실행 흐름으로 돌아오기 위해 glibc와 architecture가 내부적으로 추가한 구현 세부사항이다.

`clock_nanosleep()`은 약 0.345초가 남은 시점에 `SIGINT`로 중단됐다. `ERESTART_RESTARTBLOCK`은 kernel 내부의 system call 재시작 관련 상태이며, 이번 단계에서는 signal이 blocking system call을 중단할 수 있다는 점을 확인했다.

`Ctrl-C`는 terminal의 foreground process group에 전달되므로 `signal-demo`뿐 아니라 `strace`도 `SIGINT`를 받아 추적 대상에서 detach됐다. 특정 tracee에만 signal을 보내려면 다른 terminal에서 PID를 지정해 `kill -INT PID`를 사용할 수 있다.

## Linux System Software 직무와의 연결

Daemon과 server는 `SIGTERM`을 정상 종료 요청으로 받아 반복 작업을 중단하고 file descriptor, socket, log, thread 등의 자원을 정리해야 한다. Handler에서 복잡한 작업을 피하고 main 실행 흐름에 종료 요청을 전달하는 구조는 `sysmon-agent`의 graceful shutdown에도 그대로 사용된다.

## 학습 정리

### Learned

- Signal은 프로세스에 전달되는 비동기적인 사건 알림이다.
- `sigaction()`으로 signal disposition을 handler 실행으로 변경할 수 있다.
- Handler에서는 async-signal-safe하지 않은 작업을 피해야 한다.
- Handler가 flag만 변경하고 main이 실제 종료 작업을 수행하도록 분리했다.
- Handler는 `sigaction()` 등록이 성공한 이후부터 적용된다.
- Signal은 `sleep()` 같은 blocking system call을 중단할 수 있다.
- `SIGKILL`과 `SIGSTOP`은 handler를 등록하거나 무시할 수 없다.

### APIs

- `sigaction()`
- `sigemptyset()`
- `getpid()`
- `sleep()`

### Commands

- `man 7 signal`
- `man 2 sigaction`
- `man 7 signal-safety`
- `kill -TERM PID`
- `kill -KILL PID`
- `strace -e trace=rt_sigaction,clock_nanosleep`

### Mistakes

- `getpid()`가 실패할 수 있다고 생각해 불필요한 `-1` 검사를 추가했다.
- `sigemptyset()`의 실패 반환값 `-1`을 그대로 main에서 반환하려 했다.
- Handler 등록 직후를 기다리지 않고 너무 빨리 `SIGTERM`을 보내 기본 동작으로 종료됐다.
- 처음에는 signal을 받았다는 메시지를 signal로 직접 종료됐다는 의미로 표현했다.

### Next

- Signal로 중단된 system call과 `EINTR` 처리 방식을 학습한다.
- IPC를 통해 프로세스 사이에 데이터를 전달한다.

