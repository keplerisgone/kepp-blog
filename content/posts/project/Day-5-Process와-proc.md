---
title: "Day 5: Process와 /proc 관찰하기"
date: "2026-08-07T20:27:14+09:00"
lastmod: "2026-08-07T20:27:14+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 Linux 프로세스의 PID와 PPID를 직접 출력하고, `ps`, `/proc`, `strace`를 이용해 실행 중인 프로세스의 상태를 관찰하는 것이다.

Day 2부터 Day 4까지는 파일 입출력과 파일 메타데이터를 다뤘다. 이번에는 `fork()`를 사용하기 전에 Linux가 실행 중인 프로그램을 어떤 단위로 관리하는지 확인했다.

구현한 프로그램은 자신의 PID와 PPID를 출력하고, 관찰할 시간을 확보하기 위해 일정 시간 동안 `sleep()`한다.

```bash
./process-info
```

실행 결과는 다음과 같다.

```text
PID: 294980
PPID: 293973
Observing for 60 seconds...
finish observing
```

## 기본 이론

### 프로그램과 프로세스

프로그램은 디스크에 저장된 실행 파일이다. 프로세스는 커널이 해당 프로그램을 실행하기 위해 만든 실행 중인 인스턴스다.

```text
실행 파일 하나
├─ process-info 프로세스 A
├─ process-info 프로세스 B
└─ process-info 프로세스 C
```

같은 실행 파일을 여러 번 실행해도 각 프로세스는 서로 다른 PID, 주소 공간, 실행 상태, `/proc/[pid]` 디렉터리를 가진다.

### PID와 PPID

PID는 Process ID로 개별 프로세스를 식별한다.

PPID는 Parent Process ID로 현재 프로세스를 만든 부모 프로세스의 PID다.

터미널에서 프로그램을 실행하면 일반적으로 shell이 부모 프로세스가 된다.

```text
terminal
└─ shell PID 293973
   └─ process-info PID 294980
                       PPID 293973
```

현재 프로세스와 부모 프로세스의 PID는 다음 함수로 가져온다.

```c
getpid()
getppid()
```

두 함수는 `pid_t`를 반환하며 실패하지 않는다. 커널이 현재 프로세스의 관리 정보에서 이미 존재하는 값을 반환하기 때문이다.

`pid_t`가 반드시 `int`와 같은 타입이라고 가정하지 않고 `intmax_t`로 변환한 뒤 `%jd`로 출력했다.

### PID, PGID, SID

프로세스를 관찰하면서 PID 외에 PGID와 SID도 확인했다.

```text
PID  → 프로세스 하나
PGID → 하나의 job에 속한 프로세스 묶음
SID  → 여러 process group을 포함하는 session
```

Process group은 shell이 하나의 job을 관리하기 위한 단위다. pipeline에서는 여러 프로세스가 하나의 PGID를 공유할 수 있다.

```bash
cat file.txt | grep Linux | less
```

```text
cat  ─┐
grep ─┼─→ 같은 process group
less ─┘
```

terminal에서 `Ctrl-C`를 누르면 foreground process 하나가 아니라 foreground process group 전체에 `SIGINT`가 전달된다.

Session은 하나 이상의 process group을 묶는 더 큰 단위다. 일반적으로 하나의 로그인 또는 terminal 환경과 연결된다.

실제로 다음 결과를 확인했다.

```text
PID    PPID    PGID     SID STAT COMMAND
295366 293973  295366 293973 S+   process-info
```

이를 구조로 표현하면 다음과 같다.

```text
Session 293973
│
├─ shell
│  └─ PID 293973
│
└─ foreground Process Group 295366
   └─ process-info
      ├─ PID  295366
      ├─ PPID 293973
      ├─ PGID 295366
      └─ SID  293973
```

프로세스가 하나뿐인 job에서는 해당 프로세스가 process group leader가 되어 PID와 PGID가 같을 수 있다.

### Process state

`ps`의 `STAT` 필드에서 프로세스 상태를 확인했다.

```text
S → interruptible sleep
+ → foreground process group
N → nice 값이 양수인 낮은 우선순위
```

foreground에서 실행한 `process-info`는 다음처럼 표시됐다.

```text
S+
```

`sleep()`으로 대기하고 있어 `S` 상태이며 terminal의 foreground job이므로 `+`가 표시됐다.

zsh에서 background로 실행한 프로세스는 다음처럼 표시됐다.

```text
SN
```

`S`는 sleeping 상태이고 `N`은 프로세스의 nice 값이 양수임을 뜻한다. zsh의 `BG_NICE` 옵션이 background job의 우선순위를 낮출 수 있다.

### sleep()

`sleep()`은 지정한 초 동안 호출한 프로세스를 대기 상태로 만든다.

```c
sleep(60);
```

반환값의 의미는 다음과 같다.

```text
0
→ 지정한 시간을 모두 기다림

0보다 큰 값
→ signal로 일찍 깨어남
→ 반환값은 남은 초
```

`sleep()`의 반환 타입은 `unsigned int`다. 따라서 반환값을 출력할 때 `%d`가 아니라 `%u`를 사용해야 한다.

### /proc 가상 파일시스템

`/proc`는 디스크에 저장된 일반 파일시스템이 아니라 커널이 프로세스와 시스템 정보를 제공하는 가상 파일시스템이다.

PID가 295366인 프로세스는 다음 디렉터리에서 관찰할 수 있다.

```text
/proc/295366
```

프로세스가 종료되면 해당 디렉터리도 사라진다.

`sysmon-agent`에서 사용할 다음 파일들도 `/proc`에 있다.

```text
/proc/stat
/proc/meminfo
/proc/[pid]
```

## VmSize와 VmRSS

`/proc/[pid]/status`에서 프로세스의 가상 메모리와 실제 RAM 상주 크기를 확인할 수 있다.

```text
VmSize: 2688 kB
VmRSS:  1692 kB
```

`VmSize`는 프로세스에 할당된 전체 가상 주소 공간의 크기다.

다음 영역이 포함될 수 있다.

- 실행 코드
- 전역 변수
- heap
- stack
- shared library
- `mmap()` 영역
- 아직 실제 RAM이 연결되지 않은 예약 영역

`VmRSS`는 Resident Set Size로, 가상 주소 공간 중 현재 실제 RAM에 올라온 페이지의 대략적인 크기다.

```text
VmSize
┌───────────────────────────────┐
│ 전체 가상 주소 공간           │
│                               │
│  ┌──────────────┐             │
│  │ VmRSS        │             │
│  │ RAM에 상주   │             │
│  └──────────────┘             │
└───────────────────────────────┘
```

`VmSize`가 크다고 해서 그만큼의 RAM을 실제로 사용하고 있다는 뜻은 아니다. 또한 `VmRSS`에는 다른 프로세스와 공유하는 페이지도 포함될 수 있으므로 여러 프로세스의 RSS를 단순히 더하면 실제 RAM 사용량과 다를 수 있다.

## process-info 구현하기

구현한 전체 코드는 다음과 같다.

```c
#include <stdint.h>
#include <stdio.h>
#include <unistd.h>

#define WAIT_SECONDS 60

int main(void){
	pid_t pid;
	pid_t ppid;

	pid = getpid();
	ppid = getppid();

	printf("PID: %jd\n", (intmax_t) pid);
	printf("PPID: %jd\n", (intmax_t) ppid);

	printf("Observing for %d seconds...\n", WAIT_SECONDS);
	unsigned s_result = sleep(WAIT_SECONDS);

	if (s_result != 0) {
		fprintf(stderr, "sleep: error - left seconds: %u\n", s_result);
		return 1;
	}

	printf("finish observing\n");

	return 0;
}
```

## 프로그램의 전체 흐름

```text
현재 PID 조회
→ 부모 PID 조회
→ PID와 PPID 출력
→ sleep()으로 관찰 시간 확보
→ sleep() 반환값 확인
→ 정상 종료
```

이번 프로그램은 명령행 인자를 사용하지 않으므로 `main()`을 다음 형태로 작성했다.

```c
int main(void)
```

`getpid()`와 `getppid()`는 실패하지 않으므로 별도의 오류 검사가 필요하지 않다. `sleep()`은 지정한 시간을 모두 기다렸는지 반환값으로 확인했다.

## 컴파일

다음 명령으로 컴파일했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -o process-info process-info.c
```

최종 검사에서는 format signedness 경고도 활성화했다.

```bash
gcc -Wall -Wextra -Wpedantic -Wformat-signedness -g \
    -o process-info process-info.c
```

최종 컴파일 결과는 warning 0, error 0이었다.

## ps로 프로세스 관찰하기

프로그램이 `sleep()` 중일 때 다른 tmux pane에서 다음 명령을 실행했다.

```bash
ps -o pid,ppid,pgid,sid,stat,comm -p 295366
```

결과는 다음과 같았다.

```text
PID    PPID    PGID     SID STAT COMMAND
295366 293973  295366 293973 S+   process-info
```

프로그램이 출력한 PID와 PPID가 `ps`의 결과와 일치했다.

## /proc/[pid]/status 관찰하기

다음 명령으로 주요 상태를 확인했다.

```bash
grep -E '^(Name|State|Pid|PPid|Threads|VmSize|VmRSS):' \
    /proc/295366/status
```

결과는 다음과 같았다.

```text
Name:   process-info
State:  S (sleeping)
Pid:    295366
PPid:   293973
VmSize:     2688 kB
VmRSS:      1692 kB
Threads:        1
```

main thread 하나만 실행 중이므로 `Threads`는 1이었다.

## 표준 FD와 pseudo-terminal 관찰하기

`/proc/[pid]/fd`에서 프로세스가 열어둔 file descriptor를 확인했다.

```bash
ls -l /proc/296171/fd
```

결과는 다음과 같았다.

```text
0 -> /dev/pts/7
1 -> /dev/pts/7
2 -> /dev/pts/7
```

각 FD의 의미는 다음과 같다.

```text
0 → stdin
1 → stdout
2 → stderr
```

프로세스가 shell로부터 표준 FD를 물려받았으며, 세 FD 모두 현재 tmux pane의 pseudo-terminal을 가리키고 있었다.

실행 파일과 작업 디렉터리도 확인했다.

```bash
readlink /proc/296171/exe
readlink /proc/296171/cwd
```

```text
/proc/296171/exe
→ /home/kepp/project/linux-system-practice/file-io/process-info

/proc/296171/cwd
→ /home/kepp/project/linux-system-practice/file-io
```

`exe`는 실행 중인 프로그램 파일을, `cwd`는 프로세스의 현재 작업 디렉터리를 나타낸다.

프로세스가 종료된 뒤 `/proc/296171` 디렉터리가 사라지는 것도 확인했다.

## 같은 프로그램을 동시에 실행하기

같은 프로그램을 background에서 동시에 두 번 실행했다.

```bash
./process-info &
process_a=$!

./process-info &
process_b=$!
```

shell의 `$!`에는 가장 최근에 실행한 background job의 PID가 저장된다.

```bash
printf 'A=%s B=%s\n' "$process_a" "$process_b"
```

```text
A=296465 B=296501
```

두 프로세스를 함께 조회했다.

```bash
ps -o pid,ppid,pgid,sid,stat,comm \
    -p "$process_a,$process_b"
```

```text
PID    PPID    PGID     SID STAT COMMAND
296465 293973  296465 293973 SN   process-info
296501 293973  296501 293973 SN   process-info
```

두 프로세스는 같은 실행 파일과 부모 shell을 가졌지만 PID와 PGID는 서로 달랐다.

```text
같은 exe
같은 PPID
같은 SID
서로 다른 PID
서로 다른 PGID
서로 다른 /proc/[pid]
```

## strace로 시스템 콜 확인하기

다음 명령으로 PID 조회와 sleep에 사용되는 시스템 콜을 관찰했다.

```bash
strace -e trace=getpid,getppid,nanosleep,clock_nanosleep \
    ./process-info
```

관련 결과는 다음과 같았다.

```text
getpid()                                = 296777
getppid()                               = 296774
PID: 296777
PPID: 296774
Observing for 60 seconds...
clock_nanosleep(CLOCK_REALTIME, 0,
    {tv_sec=60, tv_nsec=0},
    0x7ffccadeafd0)                      = 0
finish observing
+++ exited with 0 +++
```

코드에서는 `sleep()`을 호출했지만 실제로는 `clock_nanosleep()` 시스템 콜이 나타났다. libc의 `sleep()`이 더 세밀한 시간 단위의 시스템 콜을 이용해 구현될 수 있기 때문이다.

`strace`로 실행했을 때 PPID는 평소 shell PID와 달랐다.

```text
shell
└─ strace PID 296774
   └─ process-info PID 296777
```

`strace`가 프로그램의 직접 부모가 되었기 때문에 `getppid()`가 `strace`의 PID를 반환했다.

## 구현 중 발생한 문제

### pid_t 출력 형식

`pid_t`가 반드시 `int`라고 가정하지 않고 `intmax_t`로 변환한 뒤 `%jd`로 출력했다. Day 4에서 구조체 필드의 실제 타입과 format string을 맞춘 경험을 다시 적용했다.

### sleep() 반환값의 format string

처음에는 `unsigned int` 타입인 `sleep()`의 반환값을 `%d`로 출력했다.

```text
unsigned int → %u
int          → %d
```

기본 컴파일 옵션에서는 경고가 나오지 않았지만 `-Wformat-signedness`를 추가해 이런 signedness 문제를 더 엄격하게 검사할 수 있다는 것을 확인했다.

### 60초의 관찰 시간

기본 동작을 확인할 때는 60초가 너무 길어 3초로 줄여 테스트했다. `/proc`와 `ps`를 다른 pane에서 관찰할 때는 다시 시간을 늘려 사용했다.

## 남은 개선 사항

현재 프로그램은 PID와 PPID만 직접 출력하고 나머지 정보는 외부 도구로 관찰한다.

향후 다음 정보를 프로그램에서 직접 조회할 수 있다.

- process group ID
- session ID
- 사용자 및 그룹 ID
- 현재 작업 디렉터리
- `/proc/self/status` 파싱
- 명령행 인자로 관찰 시간 지정

다음 단계에서는 `fork()`를 사용해 현재 프로세스가 직접 자식 프로세스를 생성한다.

```text
현재 프로세스
→ fork()
→ 부모 프로세스와 자식 프로세스
```

이후 `wait()`로 자식 프로세스의 종료를 회수하고, `exec()`로 현재 프로세스가 다른 프로그램을 실행하는 흐름까지 확장한다.

이번 실습에서는 프로그램과 프로세스의 차이, PID·PPID·PGID·SID의 역할, `/proc/[pid]`의 구조를 직접 관찰했다.

또한 같은 실행 파일을 여러 번 실행하더라도 커널은 각각을 독립적인 프로세스로 관리하며, 프로세스가 종료되면 해당 `/proc/[pid]` 디렉터리도 사라진다는 것을 확인했다.
