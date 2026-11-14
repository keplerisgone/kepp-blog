---
title: "Day 6: fork와 waitpid"
date: "2026-08-07T20:27:14+09:00"
lastmod: "2026-08-07T20:27:14+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 `fork()`로 자식 프로세스를 만들고, 부모 프로세스가 `waitpid()`로 자식의 종료 상태를 회수하는 과정을 이해하는 것이다.

Day 5에서는 이미 존재하는 프로세스의 PID와 PPID를 출력하고 `/proc`에서 상태를 관찰했다. 이번에는 현재 프로세스가 직접 새로운 자식 프로세스를 생성했다.

```text
기존 프로세스
    ↓ fork()
부모 프로세스 + 자식 프로세스
    ↓
자식 종료
    ↓ waitpid()
부모가 종료 상태 회수
```

## fork()

`fork()`는 호출한 프로세스를 바탕으로 자식 프로세스를 생성한다.

```c
pid_t pid = fork();
```

한 번 호출하지만 부모와 자식에서 각각 반환되므로 반환값의 의미가 다르다.

```text
부모 프로세스 → 자식의 PID 반환
자식 프로세스 → 0 반환
실패           → -1 반환, 자식은 생성되지 않음
```

반환값을 이용하면 같은 코드 안에서 부모와 자식의 실행 경로를 구분할 수 있다.

```text
pid == -1 → fork 실패
pid == 0  → 자식 프로세스
pid > 0   → 부모 프로세스
```

부모가 받은 값은 새로 생성된 자식의 PID이고, 자식이 받은 값은 분기를 위한 특별한 값 0이다.

## 부모와 자식의 실행 순서

`fork()` 이후에는 부모와 자식이 각각 독립적으로 스케줄링된다. 어느 프로세스가 먼저 실행될지는 커널 스케줄러가 결정하므로 출력 순서는 보장되지 않는다.

```text
가능한 순서 A
Parent 출력
Child 출력

가능한 순서 B
Child 출력
Parent 출력
```

소스 코드에서 부모 분기를 먼저 작성했다고 해서 부모의 출력이 항상 먼저 나오는 것은 아니다.

## 부모가 먼저 종료된 경우

처음에는 부모가 자식을 기다리지 않고 바로 종료하도록 구현했다. 여러 번 실행하자 자식의 PPID가 원래 부모 PID가 아닌 `1269`로 바뀌는 경우가 나타났다.

```text
Parent: PID=319098, child PID=319099
Child: PID=319099, PPID=1269, fork_result=0
```

확인 결과 PID 1269는 `systemd`였다. 부모가 자식보다 먼저 종료하면서 자식이 고아 프로세스가 되었고, 시스템의 subreaper 역할을 하는 프로세스에 다시 연결된 것이다.

```text
원래 관계
fork-demo 부모
└─ fork-demo 자식

부모가 먼저 종료
systemd
└─ fork-demo 자식
```

이 현상을 통해 자식이 `getppid()`를 호출하는 시점에 따라 PPID가 달라질 수 있다는 것을 확인했다.

## waitpid()

부모가 특정 자식의 상태 변화를 기다리고 종료 정보를 회수하기 위해 `waitpid()`를 사용했다.

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

이번 실습에서는 다음과 같이 호출했다.

```c
waitpid(pid, &status, 0);
```

각 인자의 의미는 다음과 같다.

```text
pid     → 기다릴 자식의 PID
status  → 자식의 상태 정보가 저장될 int 객체
options → 0이면 대상 자식의 종료를 blocking 방식으로 기다림
```

반환값은 다음과 같다.

```text
성공 → 상태가 변한 자식의 PID
실패 → -1, errno 설정
0    → WNOHANG을 사용했고 아직 보고할 상태 변화가 없을 때
```

이번 코드처럼 `options`가 0이면 자식이 종료할 때까지 대기하므로 성공 시 자식 PID를 반환한다.

## status와 상태 매크로

`waitpid()`의 반환값은 자식의 종료 코드가 아니다. 자식의 PID를 반환하고, 종료에 관한 정보는 `status`에 인코딩한다.

따라서 `status == 7`처럼 직접 비교하지 않고 `<sys/wait.h>`의 매크로로 해석해야 한다.

```c
if (WIFEXITED(status)) {
	int exit_code = WEXITSTATUS(status);
}
```

`WIFEXITED(status)`는 자식이 `return`, `exit()`, `_exit()`처럼 정상적인 방식으로 종료했는지 확인한다.

자식이 정상 종료한 경우에만 `WEXITSTATUS(status)`로 종료 코드를 꺼내야 한다. 시그널 때문에 종료된 경우에는 `WIFEXITED()`가 거짓이다.

관련 매크로는 다음과 같다.

```text
WIFEXITED(status)   → 정상 종료 여부
WEXITSTATUS(status) → 정상 종료 코드
WIFSIGNALED(status) → signal에 의한 종료 여부
WTERMSIG(status)    → 종료 원인이 된 signal 번호
```

시그널에 의한 종료 처리는 이후 Signal 학습에서 추가한다.

## fork-demo 구현

최종 코드는 다음과 같다.

```c
#include <stdint.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
	pid_t pid;

	pid = fork();

	if (pid == -1) {
		perror("fork");
		return 1;
	}

	if (pid == 0) {
		// Child
		pid_t child_ppid = getppid();
		pid_t child_pid = getpid();
		printf("Child: PID=%jd, PPID=%jd, fork_result=%jd\n", (intmax_t) child_pid, (intmax_t) child_ppid, (intmax_t) pid);
		sleep(1);
		printf("Child: exiting with status 7\n");
		return 7;
	}

	// Parent
	pid_t parent = getpid();
	printf("Parent: PID=%jd, child PID=%jd\n", (intmax_t) parent, (intmax_t) pid);
	int status;
	pid_t waited_pid = waitpid(pid, &status, 0);
	if (waited_pid == -1) {
		perror("waitpid");
		return 1;
	}
	if (WIFEXITED(status)) {
		int exit_code = WEXITSTATUS(status);
		printf("Parent: child %jd exited with status %d\n", (intmax_t) pid, exit_code);
	}

	return 0;
}
```

## early return으로 분기 정리하기

처음에는 `if`, `else if`, `else`로 실패·부모·자식 분기를 모두 연결했다. 동작에는 문제가 없었지만 구조가 복잡하게 느껴졌다.

자식 분기에서 작업을 마친 뒤 바로 `return 7`을 사용하도록 재배치했다.

```text
fork 실패 → 오류를 출력하고 즉시 종료
자식       → 자식 작업을 마치고 즉시 종료
그 아래    → 부모만 도달하는 코드
```

이 방식으로 불필요한 `else`와 중첩을 제거했고, 코드의 어느 부분이 부모와 자식에서 실행되는지 더 쉽게 읽을 수 있었다.

## 컴파일과 실행

다음 명령으로 컴파일했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -o fork-demo fork-demo.c
```

warning과 error 없이 컴파일됐다.

실행 결과는 다음과 같았다.

```text
Parent: PID=325861, child PID=325862
Child: PID=325862, PPID=325861, fork_result=0
Child: exiting with status 7
Parent: child 325862 exited with status 7
```

shell에서 확인한 `fork-demo` 자체의 종료 코드는 0이었다.

```bash
echo $?
```

```text
0
```

자식은 7을 반환했지만 부모는 그 값을 관찰한 뒤 0을 반환했다. shell이 보여주는 `$?`는 직접 실행한 부모 `fork-demo` 프로세스의 종료 코드다.

## strace -f로 부모와 자식 추적하기

일반적인 `strace`는 추적 대상이 만든 자식까지 자동으로 따라가지 않는다. `-f` 옵션을 사용해 부모와 자식을 함께 추적했다.

```bash
strace -f \
    -e trace=clone,fork,vfork,wait4,clock_nanosleep,exit_group \
    ./fork-demo
```

주요 결과는 다음과 같았다.

```text
clone(...) = 324049
strace: Process 324049 attached
[pid 324048] wait4(324049, <unfinished ...>
[pid 324049] clock_nanosleep(... {tv_sec=1, ...}) = 0
[pid 324049] exit_group(7) = ?
[pid 324049] +++ exited with 7 +++
<... wait4 resumed>[{WIFEXITED(s) && WEXITSTATUS(s) == 7}], 0, NULL) = 324049
--- SIGCHLD {si_code=CLD_EXITED, si_pid=324049, si_status=7, ...} ---
exit_group(0) = ?
+++ exited with 0 +++
```

전체 흐름은 다음과 같다.

```text
부모가 자식 생성
→ 부모가 wait4()에서 대기
→ 자식이 clock_nanosleep()으로 1초 대기
→ 자식이 상태 7로 종료
→ 커널이 자식의 종료 상태를 보관하고 부모에게 SIGCHLD 알림
→ 부모의 wait4()가 재개되어 자식 PID와 종료 상태 회수
→ 부모가 상태 7을 출력하고 상태 0으로 종료
```

코드에서는 `fork()`를 호출했지만 `strace`에는 `clone()`이 나타났다. Linux에서 glibc의 `fork()`가 내부적으로 `clone()` 계열 시스템 콜을 이용해 구현될 수 있기 때문이다.

코드의 `waitpid()`는 Linux에서 `wait4()` 시스템 콜로 나타났다. libc API의 이름과 실제 Linux 시스템 콜의 이름이 항상 같지는 않다는 것을 다시 확인했다.

`<unfinished ...>`와 `<... resumed>`는 부모의 `wait4()` 호출이 자식의 종료를 기다리며 중단됐다가 이후 재개된 모습을 나타낸다.

## 좀비 프로세스 관찰

자식은 종료했지만 부모가 아직 `waitpid()`를 호출하지 않은 상황을 만들기 위해 부모의 `waitpid()` 직전에 일시적으로 `sleep(10)`을 넣었다.

다른 terminal에서 자식 PID를 조회했다.

```bash
ps -o pid,ppid,stat,comm,args -p 325674
```

결과는 다음과 같았다.

```text
PID     PPID   STAT COMMAND    COMMAND
325674  325673 Z+   fork-demo  [fork-demo] <defunct>
```

각 표시는 다음을 의미한다.

```text
Z         → zombie 상태
+         → terminal의 foreground process group에 속함
<defunct> → 실행은 끝났지만 종료 정보가 아직 회수되지 않음
```

좀비 프로세스는 계속 실행되는 프로세스가 아니다. 자식의 실행은 이미 끝났지만 부모가 확인할 종료 정보와 일부 관리 항목이 커널의 process table에 남아 있는 상태다.

부모가 `waitpid()`를 호출한 뒤 같은 명령을 다시 실행하자 해당 PID가 사라졌다.

```text
자식 종료
→ 종료 정보가 남아 zombie 상태
→ 부모가 waitpid() 호출
→ 종료 정보 회수
→ process table에서 자식 항목 제거
```

관찰이 끝난 뒤 임시로 추가한 부모의 `sleep(10)`은 제거했다.

## 구현 중 확인한 문제

### 출력 끝의 `%` 문자

처음 자식의 출력 끝에 개행 문자가 없어서 zsh prompt의 `%`가 같은 줄에 붙어 보였다. format string 끝에 `\n`을 추가해 해결했다.

### perror()의 문자열

`perror()`는 전달한 문자열 뒤에 콜론과 `errno` 메시지를 붙인다.

```c
perror("fork");
perror("waitpid");
```

따라서 문자열에 별도의 콜론을 넣지 않았다.

### waitpid()와 종료 코드의 혼동

`waitpid()`의 반환값을 자식의 종료 코드와 혼동할 수 있었다.

```text
waitpid() 반환값 → 자식 PID
status           → 인코딩된 상태 정보
WEXITSTATUS()    → 정상 종료한 자식의 종료 코드
```

### 정상 종료와 signal 종료

`WIFEXITED()`는 프로세스가 단순히 사라졌는지를 확인하는 것이 아니라 정상적인 종료 방식을 사용했는지를 확인한다. signal로 종료되었다면 `WIFSIGNALED()`과 `WTERMSIG()`을 사용해야 한다.

이 부분은 Signal 학습 때 구현할 TODO로 남긴다.

## Linux System Software 직무와의 연결

실제 Linux 시스템 프로그램은 worker process를 만들거나 외부 프로그램을 실행하고, 종료 상태를 회수해야 한다. 자식을 회수하지 않으면 장시간 실행되는 daemon에 zombie가 누적될 수 있다.

`fork()`, `waitpid()`, PID 관계와 종료 상태 해석은 process manager, shell, daemon, server를 구현할 때 필요한 기본기다.

## 학습 정리

### Learned

- `fork()` 한 번의 호출 이후 부모와 자식 두 실행 흐름이 생긴다.
- 부모와 자식의 실행 순서는 보장되지 않는다.
- 부모가 먼저 종료하면 자식의 PPID가 다른 reaper로 바뀔 수 있다.
- `waitpid()`는 자식의 상태를 기다리고 종료 정보를 회수한다.
- 종료했지만 아직 회수되지 않은 자식은 zombie 상태로 남는다.
- early return으로 부모와 자식의 분기를 단순하게 정리할 수 있다.

### APIs

- `fork()`
- `getpid()`
- `getppid()`
- `waitpid()`
- `sleep()`
- `WIFEXITED()`
- `WEXITSTATUS()`

### Commands

- `gcc -Wall -Wextra -Wpedantic -g`
- `ps -o pid,ppid,stat,comm,args`
- `strace -f`

### Mistakes

- 자식 출력의 마지막에 개행 문자를 빠뜨렸다.
- `waitpid()`의 반환값과 `status`에 저장되는 값을 혼동했다.
- `WNOHANG`을 정상 종료 여부를 확인하는 매크로로 생각했다.
- 부모가 자식을 기다리지 않으면 PPID가 실행 도중 바뀔 수 있다는 점을 직접 확인했다.

### Next

- `exec()` 계열 함수로 현재 프로세스의 프로그램 이미지를 교체한다.
- `fork()`와 `exec()`를 조합한다.
- `waitpid()`로 `exec()`한 자식의 종료 상태를 회수한다.

