---
title: "Day 9: Pipe IPC와 EOF"
date: "2026-08-16T15:40:12+09:00"
lastmod: "2026-08-16T15:40:12+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 익명 pipe를 이용해 부모 프로세스에서 자식 프로세스로 데이터를 전달하고, `fork()` 이후 file descriptor의 상속과 pipe의 EOF 조건을 이해하는 것이다.

Day 8에서 signal이 blocking system call을 중단할 수 있다는 점을 관찰했다. 이번에는 `read()`가 `EINTR`로 중단됐을 때 재시도하는 처리도 추가했다.

```text
Parent
pipe write end
      │
      │ write()
      ▼
Kernel pipe buffer
      │
      │ read()
      ▼
pipe read end
Child
```

## IPC와 익명 pipe

IPC는 Inter-Process Communication의 약자로 서로 다른 프로세스가 데이터를 주고받는 방법을 의미한다.

익명 pipe는 kernel이 관리하는 단방향 byte stream이다.

```c
int pipefd[2];
pipe(pipefd);
```

`pipe()`가 성공하면 배열에 두 file descriptor를 저장한다.

```text
pipefd[0] → read end
pipefd[1] → write end
```

반환값은 다음과 같다.

```text
성공 → 0
실패 → -1, errno 설정
```

Pipe는 message 단위를 보존하는 구조가 아니라 byte stream이다. 한 번의 `write()`가 반드시 한 번의 동일한 크기 `read()`와 대응한다고 가정하면 안 된다.

## pipe() 이후 FD 구조

프로세스는 일반적으로 표준 FD 0, 1, 2를 이미 가지고 있다.

```text
FD 0 → stdin
FD 1 → stdout
FD 2 → stderr
```

`pipe()`가 다음으로 사용 가능한 번호 3과 4를 반환했다고 가정하면 구조는 다음과 같다.

```text
Process FD table

FD 3: read end  ─┐
                 ├─ Kernel pipe buffer
FD 4: write end ─┘
```

FD 번호 3과 4는 예시일 뿐이며 항상 고정된 값이라고 가정할 수 없다.

## fork()와 FD 상속

`pipe()`를 생성한 다음 `fork()`를 호출했다. 자식은 부모의 FD table 항목을 물려받으므로 fork 직후 부모와 자식 모두 pipe 양쪽 끝을 가지고 있다.

```text
Parent                         Child
FD 3 → pipe read end           FD 3 → pipe read end
FD 4 → pipe write end          FD 4 → pipe write end
```

FD 번호는 프로세스별 table의 index다. 부모와 자식의 FD 4는 서로 다른 table 항목이지만 같은 kernel pipe의 write end를 참조한다.

따라서 자식이 자신의 FD 4를 닫아도 부모의 FD 4는 닫히지 않는다.

참조 개수의 변화는 다음과 같다.

| 시점 | Read end 참조 | Write end 참조 |
|---|---:|---:|
| `pipe()` 직후 | 1 | 1 |
| `fork()` 직후 | 2 | 2 |
| 사용하지 않는 끝을 닫은 후 | 1 | 1 |
| 부모가 write end를 닫은 후 | 1 | 0 |
| 자식이 read end를 닫은 후 | 0 | 0 |

## 사용하지 않는 끝 닫기

이번 프로그램의 데이터 방향은 부모에서 자식이다.

```text
Parent ──write──> Child
```

따라서 부모는 read end를, 자식은 write end를 닫았다.

```text
Parent → close(pipefd[0])
Child  → close(pipefd[1])
```

정리한 뒤에는 다음 구조만 남는다.

```text
Parent pipefd[1]
        │
        ▼
Kernel pipe buffer
        │
        ▼
Child pipefd[0]
```

자식이 사용하지 않는 write end를 닫지 않으면 부모가 자신의 write end를 닫아도 pipe에는 writer가 하나 남아 있다고 판단된다. 이 경우 자식은 buffer를 모두 읽은 뒤에도 EOF를 받지 못하고 자신이 보유한 writer를 기다리며 block될 수 있다.

## Pipe의 blocking과 EOF

기본 blocking mode에서 빈 pipe에 write end가 하나 이상 열려 있으면 `read()`는 앞으로 데이터가 들어올 가능성이 있으므로 대기한다.

```text
buffer가 비어 있음
+ write end가 열려 있음
→ read() block
```

모든 write end가 닫혀도 buffer에 데이터가 남아 있다면 `read()`는 그 데이터를 먼저 반환한다. 남은 데이터를 모두 읽은 뒤 다음 조건에서 EOF가 발생한다.

```text
모든 write end가 닫힘
+ pipe buffer가 비어 있음
→ read() == 0
→ EOF
```

`read()` 반환값은 다음처럼 해석한다.

```text
0보다 큼 → 실제 읽은 byte 수
0        → EOF
-1       → 오류, errno 확인
```

EOF는 실패가 아니라 stream의 정상적인 끝이다.

## EINTR

Blocking system call 실행 중 처리 가능한 signal이 전달되면 호출이 중단될 수 있다.

```text
반환값 == -1
errno == EINTR
→ signal 처리로 system call이 중단됨
```

자식의 `read()`에서는 `EINTR`을 실제 I/O 오류와 구분해 다시 시도했다.

```text
read() > 0               → 데이터 처리
read() == 0              → EOF, 반복 종료
read() == -1 && EINTR    → 다시 read
그 외 -1                → perror 후 오류 종료
```

`errno`와 `EINTR`을 사용하기 위해 user-space 표준 인터페이스인 `<errno.h>`를 포함했다. 처음에는 `<asm-generic/errno-base.h>`도 추가했지만, 이는 Linux kernel/UAPI 내부 계층의 헤더이므로 직접 include하지 않고 제거했다.

## pipe-demo 구현

최종 코드는 다음과 같다.

```c
#include <stdio.h>
#include <stdint.h>
#include <unistd.h>
#include <sys/wait.h>
#include <errno.h>

int main(void) {
	int pipefd[2];
	if (pipe(pipefd) == -1) {
		perror("pipe");
		return 1;
	}

	pid_t pid = fork();

	if (pid == -1) {
		perror("fork");
		close(pipefd[0]);
		close(pipefd[1]);
		return 1;
	}

	// child
	if (pid == 0) {
		close(pipefd[1]);
		// read from pipe
		char buffer[128];
		ssize_t bytes_read = read(pipefd[0], buffer, sizeof(buffer)-1);
		while (1) {
			if (bytes_read > 0) {
				buffer[bytes_read] = '\0';
				printf("Child received: %s\n", buffer);
				bytes_read = read(pipefd[0], buffer, sizeof(buffer)-1);
			} else if (bytes_read == 0) {
				printf("Child: EOF\n");
				break;
			} else if (bytes_read == -1 && errno == EINTR) {
				bytes_read = read(pipefd[0], buffer, sizeof(buffer)-1);
			} else {
				perror("read");
				close(pipefd[0]);
				return 1;
			}
		}
		close(pipefd[0]);
		return 0;
	}

	// parent
	int result = 0;
	close(pipefd[0]);
	const char message[] = "hello from parent";
	ssize_t bytes_write = write(pipefd[1], message, sizeof(message)-1);
	if (bytes_write == -1) {
		perror("write");
		result = 1;
	} else if (bytes_write < (ssize_t) (sizeof(message)-1)) {
		fprintf(stderr, "partial write\n");
		result = 1;
	}
	close(pipefd[1]);

	int status;
	pid_t waited_pid = waitpid(pid, &status, 0);
	if (waited_pid == -1) {
		perror("waitpid");
		return 1;
	}
	if (WIFEXITED(status)) {
		int child_result = WEXITSTATUS(status);
		printf("Parent: child %jd exited with status %d\n", (intmax_t) pid, child_result);
		result = (child_result != 0) ? child_result : result;
	} else if (WIFSIGNALED(status)) {
		int term_signal = WTERMSIG(status);
		printf("Parent: child %jd terminated by signal %d\n", (intmax_t) pid, term_signal);
		result = 128 + term_signal;
	}

	return result;
}
```

## 프로그램 실행 결과

다음 옵션으로 컴파일했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -o pipe-demo pipe-demo.c
```

Warning과 error 없이 컴파일됐다.

실행 결과는 다음과 같았다.

```text
Child received: hello from parent
Child: EOF
Parent: child 10 exited with status 0
```

프로그램의 최종 종료 상태도 0이었다.

## 구현 중 발생한 문제

### size_t와 ssize_t

처음에는 `read()` 반환값을 `size_t`에 저장했다.

```text
size_t  → unsigned, byte 크기 표현
ssize_t → signed, byte 수 또는 -1 오류 표현
```

`read()`와 `write()`는 실패 값 `-1`을 반환해야 하므로 결과를 `ssize_t`에 저장해야 한다.

### Short read를 오류로 판단

처음에는 읽은 byte 수가 buffer 크기보다 작으면 오류라고 판단했다.

```text
read 요청: 127 bytes
실제 데이터: 17 bytes
read 반환: 17
```

`read()`는 요청한 buffer를 반드시 가득 채우지 않는다. 17은 정상적인 성공 반환값이다.

### perror("read"): Success

성공한 `read()`를 잘못 오류 분기로 보내 `perror()`를 호출하자 다음 메시지가 나타났다.

```text
read: Success
```

`perror()`는 함수의 반환값을 확인하지 않고 현재 `errno`를 해석한다. 따라서 system call이 `-1`을 반환해 `errno`를 설정한 경우에만 사용해야 한다.

EOF처럼 오류가 아닌 상황의 메시지는 `fprintf()`로 출력해야 한다.

### errno를 확인하기 전 다른 system call 호출

처음에는 `read()`가 실패한 뒤 `close()`를 먼저 호출하고 `perror()`를 실행했다. 다른 system call이 `errno`를 바꿀 수 있으므로 원래 오류를 확인하려면 `read()` 실패 직후 `perror()`를 호출해야 한다.

### 중복 close와 waitpid

부모의 `write()` 오류 분기 안에서 `close()`와 `waitpid()`를 호출한 뒤 공통 경로에서도 다시 호출하는 구조가 있었다.

```text
오류 처리의 close/waitpid
→ 아래 공통 경로의 close/waitpid
→ 같은 자원과 자식을 두 번 처리
```

오류 여부는 `result`에 기록하고 cleanup과 `waitpid()`는 한 곳에서 한 번만 수행하도록 정리했다.

### 부모 오류가 자식 성공으로 사라지는 문제

부모의 `write()`가 실패해 `result = 1`이 된 뒤 자식이 0으로 종료하면 부모가 `result`를 다시 0으로 덮어쓸 수 있었다.

```c
result = (child_result != 0) ? child_result : result;
```

자식이 성공했다면 기존 부모 오류 상태를 유지하고, 자식이 실패했다면 자식의 종료 상태를 반영했다.

### waitpid 실패 후 status 사용

`waitpid()`가 실패하면 `status`는 유효한 자식 상태를 담고 있다고 볼 수 없다. 실패 후 `WIFEXITED(status)`를 호출하지 않고 즉시 오류를 반환하도록 구성했다.

## strace로 Pipe 흐름 관찰

다음 명령으로 부모와 자식을 함께 추적했다.

```bash
strace -f \
    -e trace=pipe,pipe2,clone,read,write,close,wait4 \
    ./pipe-demo
```

`main()` 실행 전 다음과 같은 ELF 파일 읽기가 먼저 나타났다.

```text
read(3, "\177ELF...", 832) = 832
close(3) = 0
```

이는 dynamic linker가 shared library 등의 ELF 파일을 읽는 과정이다. 해당 FD 3이 닫힌 뒤 같은 번호가 pipe의 read end로 재사용됐다.

Pipe 생성과 자식 생성은 다음처럼 나타났다.

```text
pipe2([3, 4], 0) = 0
clone(...) = 407302
```

소스에서는 `pipe()`와 `fork()`를 호출했지만 Linux의 실제 system call 수준에서는 `pipe2()`와 `clone()`이 나타났다.

부모의 흐름:

```text
close(3) = 0
write(4, "hello from parent", 17) = 17
close(4) = 0
wait4(407302, ... <unfinished ...>
```

부모는 read end를 닫고 17 byte를 모두 쓴 뒤 마지막 write end를 닫았다. 이후 자식 종료를 기다리며 `wait4()`에서 block됐다.

자식의 흐름:

```text
[pid 407302] close(4) = 0
[pid 407302] read(3, "hello from parent", 127) = 17
[pid 407302] read(3, "", 127) = 0
[pid 407302] close(3) = 0
[pid 407302] +++ exited with 0 +++
```

첫 `read()`는 pipe buffer의 17 byte를 반환했다. 모든 write end가 닫히고 buffer까지 비워진 뒤 두 번째 `read()`는 0을 반환해 EOF를 알렸다.

자식이 종료하자 부모의 `wait4()`가 재개됐다.

```text
<... wait4 resumed>[{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL)
    = 407302
--- SIGCHLD {si_pid=407302, si_status=0, ...} ---
```

`printf()`가 최종적으로 stdout FD 1에 대한 `write()`로 나타나는 것도 확인했다.

## 남은 개선 사항

- Parent의 partial write를 감지만 하지 않고 남은 부분부터 반복한다.
- `write()`가 `EINTR`로 중단됐을 때 안전하게 재시도한다.
- 반복문 맨 위에서 `read()`를 한 번 호출하도록 구조를 단순화한다.
- 정상 경로의 `close()` 실패 처리 정책을 정리한다.
- 예상하지 못한 자식 상태에 대한 fallback을 추가한다.

Partial write 처리는 이후 Network Programming 전에 재사용 가능한 `write_all()` 형태로 다시 구현한다.

## Linux System Software 직무와의 연결

Pipe는 shell pipeline, worker process, logger, process supervisor 등에서 사용하는 기본 IPC다. 단순히 `read()`와 `write()`를 호출하는 것보다 FD 상속, 사용하지 않는 끝의 정리, EOF 조건과 blocking을 이해해야 deadlock과 자원 누수를 피할 수 있다.

## 학습 정리

### Learned

- `pipefd[0]`은 read end, `pipefd[1]`은 write end다.
- `fork()` 이후 부모와 자식은 pipe 양쪽 FD를 모두 물려받는다.
- 사용하지 않는 pipe 끝을 닫아야 EOF와 오류가 올바르게 전달된다.
- Pipe는 kernel buffer를 사용하는 단방향 byte stream이다.
- 모든 writer가 닫히고 buffer가 비면 `read()`가 0을 반환한다.
- Short read는 오류가 아니며 `read()` 결과는 `ssize_t`에 저장한다.
- `EINTR`은 signal 때문에 system call이 중단됐음을 뜻한다.

### APIs

- `pipe()`
- `fork()`
- `read()`
- `write()`
- `close()`
- `waitpid()`
- `WIFEXITED()` / `WEXITSTATUS()`
- `WIFSIGNALED()` / `WTERMSIG()`

### Commands

- `man 2 pipe`
- `man 7 pipe`
- `strace -f -e trace=pipe,pipe2,clone,read,write,close,wait4`

### Mistakes

- `read()` 반환값을 unsigned인 `size_t`에 저장했다.
- 요청한 크기보다 적게 읽은 정상적인 short read를 오류로 처리했다.
- 성공한 system call 뒤에서 `perror()`를 호출해 `Success`가 출력됐다.
- 오류 분기와 공통 경로에서 `close()`와 `waitpid()`를 중복 호출했다.
- 부모의 오류 상태를 자식의 성공 상태 0으로 덮어쓸 수 있었다.
- User-space 코드에서 kernel 내부 계층의 errno 헤더를 직접 포함했다.

### Next

- Named pipe인 FIFO로 관련 없는 두 프로세스가 통신한다.
- IPC 기초 이후 GDB를 이용한 source-level debugging을 시작한다.

