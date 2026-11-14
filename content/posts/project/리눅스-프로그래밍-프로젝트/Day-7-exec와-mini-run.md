---
title: "Day 7: exec와 mini-run"
date: "2026-08-13T22:03:28+09:00"
lastmod: "2026-08-13T22:03:28+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-7-exec와-mini-run/"
weight: 7
ShowToc: true
---

이번 실습의 목표는 `exec()` 계열 함수로 현재 프로세스의 프로그램 이미지를 교체하고, Day 6에서 배운 `fork()`와 `waitpid()`를 결합해 간단한 명령 실행기 `mini-run`을 구현하는 것이다.

```text
부모 mini-run
    ↓ fork()
자식 mini-run
    ↓ execvp()
사용자가 지정한 프로그램
```

## fork와 exec의 차이

`fork()`와 `exec()`는 서로 다른 역할을 한다.

```text
fork() → 새로운 프로세스 생성
exec() → 현재 프로세스의 프로그램 이미지 교체
```

`exec()`는 새 프로세스를 생성하지 않는다. 기존 프로세스의 PID를 유지하면서 실행 중인 코드, 데이터, stack 등의 프로그램 이미지를 새로운 실행 파일의 것으로 교체한다.

```text
PID 100: exec-demo
        ↓ exec("sleep")
PID 100: sleep
```

## 성공한 exec는 반환하지 않는다

일반적인 함수는 실행을 마치면 호출 지점의 다음 줄로 돌아온다. `exec()`가 성공하면 기존 프로그램 이미지와 호출 stack이 새로운 프로그램의 것으로 교체되므로 기존 코드의 다음 줄로 돌아갈 수 없다.

```text
exec 성공
→ 기존 프로그램 이미지 제거
→ 새 프로그램 이미지 적재
→ 새 프로그램 실행 시작
```

실행 파일을 찾지 못하는 등의 이유로 실패하면 교체가 발생하지 않는다. 이 경우에만 `exec()`가 `-1`을 반환하고 `errno`를 설정한다.

따라서 다음 구조에서 `perror()`는 `exec()` 실패 경로다.

```c
exec...(...);
perror("exec");
```

## exec 함수 이름

`exec` 계열 함수의 이름은 인자 전달 방법과 실행 파일 검색 방법을 나타낸다.

```text
l → 인자를 list 형태로 전달
v → 인자를 vector, 즉 argv 배열로 전달
p → PATH 환경 변수에서 실행 파일 검색
e → 환경 변수 배열을 직접 전달
```

고정된 수의 인자를 넘긴 `exec-demo`에서는 `execlp()`를 사용했다. 사용자가 입력한 명령행 인자를 그대로 전달하는 `mini-run`에서는 `execvp()`를 사용했다.

## argv[0]과 NULL terminator

`argv[0]`에는 관례상 실행되는 프로그램의 이름을 넣는다.

```text
argv[0] → 프로그램 이름
argv[1] → 첫 번째 인자
...
argv[n] → NULL
```

`execlp()`는 인자의 개수를 별도로 받지 않는 가변 인자 함수다. 마지막의 `(char *)NULL`이 인자 목록의 끝을 표시한다.

```c
execlp("sleep", "sleep", "30", (char *)NULL);
```

처음에는 정수 `5`를 전달하고 NULL을 생략했다.

```text
5   → 정수
"5" → 명령행에 전달할 문자열
```

명령행 인자는 문자열이어야 하며, NULL을 생략하면 함수가 인자 목록의 끝을 알 수 없어 잘못된 메모리를 계속 해석할 수 있다.

배열을 전달하는 `execv()`와 `execvp()`에서도 배열의 마지막 원소는 NULL이어야 한다. C 프로그램에 전달되는 기존 `argv`도 `argv[argc] == NULL`을 만족한다.

## exec-demo

먼저 `fork()` 없이 현재 프로그램을 `sleep`으로 교체했다.

```c
#include <stdint.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
	pid_t pid = getpid();
	printf("pid: %jd\n", (intmax_t) pid);

	fflush(stdout);
	execlp("sleep", "sleep", "30", (char *) NULL);
	perror("execlp");

	return 1;
}
```

`fflush(stdout)`은 `exec()`로 user-space의 stdio buffer가 사라지기 전에 PID 출력을 확실히 전달하기 위해 호출했다.

백그라운드에서 실행한 뒤 `ps`로 관찰했다.

```bash
./exec-demo &
demo_pid=$!
ps -o pid,ppid,stat,comm,args -p "$demo_pid"
```

실행 환경에서 다음 결과를 확인했다.

```text
shell observed PID: 3
pid: 3
PID PPID STAT COMMAND COMMAND
3   2    SN   sleep   sleep 30
```

`exec-demo`가 출력한 PID와 shell이 관찰한 PID는 3으로 같았다. `ps`의 프로그램 이름과 인자는 `sleep 30`으로 바뀌었다. 이를 통해 PID는 유지되고 프로그램 이미지만 교체된다는 것을 확인했다.

## mini-run의 argv 전달

다음 명령을 실행할 때 기존 argv는 다음 구조다.

```bash
./mini-run echo hello Linux
```

```text
argv[0] → "./mini-run"
argv[1] → "echo"
argv[2] → "hello"
argv[3] → "Linux"
argv[4] → NULL
```

새 프로그램 `echo`에 전달할 argv는 `argv[1]`부터 시작해야 한다.

```text
&argv[1]
    ↓
"echo", "hello", "Linux", NULL
```

따라서 자식 프로세스에서 다음 형태로 호출했다.

```c
execvp(argv[1], &argv[1]);
```

첫 번째 인자는 PATH에서 검색할 명령이고, 두 번째 인자는 새 프로그램에 전달할 argv 배열이다.

## exec 실패와 _exit

`execvp()`가 실패한 경우 자식 프로세스에서 오류를 출력하고 상태 127로 종료했다.

```c
execvp(argv[1], &argv[1]);
perror(argv[1]);
_exit(127);
```

127은 shell 계열 프로그램에서 명령을 찾거나 실행하지 못했음을 나타낼 때 흔히 사용하는 종료 코드다.

`fork()` 이후 자식의 오류 경로에서는 일반적인 `exit()`나 `return` 대신 `_exit()`을 사용했다. 부모로부터 복제된 stdio buffer와 `atexit()` handler를 다시 처리하지 않고 자식 프로세스를 즉시 종료하기 위해서다.

## mini-run 구현

최종 프로그램의 흐름은 다음과 같다.

```text
명령행 인자 검사
→ fork()
→ 자식은 execvp()로 명령 실행
→ exec 실패 시 _exit(127)
→ 부모는 waitpid()로 자식 회수
→ 정상 종료 코드 또는 signal 번호 해석
→ 결과를 mini-run의 종료 코드로 전달
```

구현한 코드는 다음과 같다.

```c
#include <stdint.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	if (argc < 2) {
		fprintf(stderr, "usage: %s <command>\n", argv[0]);
		return 1;
	}

	pid_t pid = fork();

	if (pid == -1) {
		perror("fork");
		return 1;
	}

	if (pid == 0) {
		// child
		execvp(argv[1], &argv[1]);
		perror(argv[1]);
		_exit(127);
	}

	// parent
	int status;
	pid_t waited_pid = waitpid(pid, &status, 0);
	if (waited_pid == -1) {
		perror("waitpid");
		return 1;
	}
	if (WIFEXITED(status)) {
		int exit_code = WEXITSTATUS(status);
		printf("Parent: child %jd exited with status %d\n", (intmax_t) pid, exit_code);
		return exit_code;
	}
	if (WIFSIGNALED(status)) {
		int signal = WTERMSIG(status);
		printf("Parent: child %jd terminated by signal %d\n", (intmax_t) pid, signal);
		return (128 + signal);
	}

	fprintf(stderr, "unexpected  child status\n");
	return 1;
}
```

## 종료 상태 전달

처음에는 부모가 자식의 상태를 출력한 뒤 항상 0을 반환했다. 이 경우 자식의 명령이 실패해도 shell에서는 `mini-run`이 성공한 것처럼 보인다.

최종 구현에서는 정상 종료한 자식의 종료 코드를 부모도 반환한다.

```text
자식 exit 0   → mini-run return 0
자식 exit 7   → mini-run return 7
exec 실패     → 자식 127, mini-run return 127
```

signal로 종료했다면 `WIFSIGNALED()`로 확인하고 `WTERMSIG()`로 signal 번호를 추출했다. shell의 일반적인 관례에 따라 `128 + signal`을 반환했다.

```text
SIGTERM 15
→ 128 + 15
→ 종료 상태 143
```

`WIFSIGNALED()`을 처음에는 `WIFEXITED()` 내부에 배치했다. 정상 종료와 signal 종료는 서로 배타적이므로 이 조건은 절대 참이 될 수 없었다. 두 조건을 같은 단계의 독립된 분기로 배치해 해결했다.

## 테스트

다음 명령을 모두 실행했다.

```bash
./mini-run echo hello Linux
./mini-run ls -l
./mini-run sh -c 'exit 7'
./mini-run nonexistent-command
./mini-run sh -c 'kill -TERM $$'
```

확인한 결과는 다음과 같다.

```text
echo                 → 0
ls                   → 0
sh -c 'exit 7'       → 7
nonexistent-command  → 127
SIGTERM 종료         → 143
```

signal 테스트에서 작은따옴표를 사용해 바깥 shell이 아닌 `mini-run`이 실행한 `sh`가 `$$`를 자신의 PID로 해석하게 했다.

## strace로 exec 관찰

다음 명령으로 부모와 자식을 함께 추적했다.

```bash
strace -f \
    -e trace=clone,execve,wait4,exit_group \
    ./mini-run echo hello
```

관찰한 흐름은 다음과 같다.

```text
mini-run 실행
→ clone()으로 자식 생성
→ 부모가 wait4()에서 대기
→ 자식이 PATH의 후보 경로마다 execve() 시도
→ 실행 가능한 echo 경로에서 execve() 성공
→ echo가 0으로 종료
→ 부모의 wait4()가 재개되어 자식 회수
```

소스에서는 libc 함수 `execvp()`를 호출했지만 system call 수준에서는 `execve()`가 나타났다.

## PATH 검색의 이유

`execvp()`의 `p`는 PATH 환경 변수에 나열된 디렉터리를 순서대로 검색한다는 의미다. 성공한 경로를 찾으면 검색을 중단한다.

```text
PATH의 첫 경로에서 execve() → ENOENT
다음 경로에서 execve()      → ENOENT
실제 실행 파일 경로          → 성공
```

경로 조회 비용이 발생하지만 사용자는 프로그램의 절대 경로를 알 필요가 없고, 프로세스마다 검색 순서와 실행 파일 우선순위를 설정할 수 있다. kernel의 `execve()`는 지정된 경로를 실행하는 기능만 제공하고 PATH 검색 정책은 libc와 shell 같은 user-space가 담당한다.

권한이 중요한 daemon에서는 신뢰할 수 없는 PATH로 인해 같은 이름의 악성 프로그램을 실행할 위험이 있다. 실행 대상이 고정돼 있다면 절대 경로나 통제된 PATH를 사용하는 편이 더 예측 가능하다.

## 남은 개선 사항

- usage를 `<COMMAND> [ARG...]` 형태로 더 정확하게 표시한다.
- `signal` 변수명을 `term_signal`처럼 더 구체적으로 변경한다.
- `unexpected  child status`의 이중 공백을 제거한다.
- Signal 학습 시 signal로 중단된 `waitpid()`와 `EINTR` 처리를 다룬다.

## Linux System Software 직무와의 연결

shell, process supervisor, daemon과 server는 자식 프로세스를 생성하고 외부 프로그램을 실행하며 종료 결과를 관리해야 한다. `fork()`·`exec()`·`waitpid()`의 조합과 신뢰할 수 있는 실행 경로 관리는 Linux process management의 핵심이다.

## 학습 정리

### Learned

- `exec()`는 새 프로세스를 생성하지 않고 현재 프로그램 이미지를 교체한다.
- 성공한 `exec()`는 기존 호출 지점으로 반환하지 않는다.
- `exec()` 전후로 PID가 유지된다.
- `argv` 배열은 NULL로 끝나며 명령행 인자는 문자열이다.
- `execvp()`는 PATH를 순서대로 검색한다.
- 부모가 자식의 정상·signal 종료 상태를 구분해 전달할 수 있다.

### APIs

- `execlp()`
- `execvp()`
- `_exit()`
- `fork()`
- `waitpid()`
- `WIFEXITED()` / `WEXITSTATUS()`
- `WIFSIGNALED()` / `WTERMSIG()`

### Commands

- `man 3 exec`
- `man 2 execve`
- `ps -o pid,ppid,stat,comm,args`
- `strace -f -e trace=clone,execve,wait4,exit_group`

### Mistakes

- `execlp()`에 문자열 `"5"` 대신 정수 `5`를 전달했다.
- 가변 인자 목록을 끝내는 `(char *)NULL`을 빠뜨렸다.
- 자식 종료 상태를 확인하고도 부모가 항상 0을 반환했다.
- 서로 배타적인 `WIFSIGNALED()`을 `WIFEXITED()` 내부에 배치했다.

### Next

- signal의 기본 동작과 signal handler를 학습한다.
- signal에 안전한 종료와 system call interruption을 관찰한다.

