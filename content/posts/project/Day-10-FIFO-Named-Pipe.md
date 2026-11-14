---
title: "Day 10: FIFO Named Pipe"
date: "2026-08-17T12:53:33+09:00"
lastmod: "2026-08-17T12:53:33+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 filesystem pathname을 가진 FIFO를 이용해 부모·자식 관계가 없는 두 프로세스가 통신하도록 만드는 것이다.

Day 9의 익명 pipe는 `fork()`로 file descriptor를 물려받은 부모와 자식이 통신했다. FIFO는 두 프로세스가 같은 pathname을 각각 `open()`해 kernel의 통신 채널에 연결된다.

```text
fifo-writer
    │
    │ write()
    ▼
Kernel FIFO buffer
    │
    │ read()
    ▼
fifo-reader
```

## 익명 pipe와 FIFO

익명 pipe와 FIFO는 모두 kernel의 pipe buffer를 사용하는 단방향 byte stream이다. 가장 큰 차이는 통신 채널을 찾는 방법이다.

```text
익명 pipe
→ pipe()가 반환한 FD 사용
→ 일반적으로 fork()로 FD 상속

FIFO
→ filesystem pathname 사용
→ 관련 없는 프로세스도 같은 이름으로 open 가능
```

FIFO pathname은 두 프로세스가 같은 통신 채널을 찾기 위한 rendezvous point 역할을 한다.

```text
fifo-reader
→ open("demo.fifo", O_RDONLY)

fifo-writer
→ open("demo.fifo", O_WRONLY)
```

## FIFO 생성과 metadata

다음 명령으로 FIFO를 생성했다.

```bash
mkfifo demo.fifo
```

`stat` 결과는 다음과 같았다.

```text
File: demo.fifo
Size: 0
Blocks: 0
IO Block: 4096 fifo
Access: (0664/prw-rw-r--)
Inode: 23466105
Links: 1
```

파일 종류를 나타내는 첫 글자 `p`와 `fifo` 표시로 FIFO임을 확인했다.

```text
- → regular file
d → directory
l → symbolic link
p → FIFO
```

`file` 명령도 named pipe로 판별했다.

```text
demo.fifo: fifo (named pipe)
```

FIFO는 filesystem에 이름, inode, permission을 가지지만 전달 데이터는 일반 파일 내용으로 저장되지 않는다.

```text
filesystem
→ demo.fifo 이름과 inode 보관

kernel memory
→ 실행 중 전달되는 FIFO 데이터 보관
```

따라서 통신 전후에도 `Size`와 `Blocks`는 0이다. `IO Block: 4096`은 pipe buffer의 용량을 의미하지 않는다.

Permission은 FIFO pathname을 `open()`할 수 있는 사용자를 결정한다. 생성 시 요청한 mode에는 현재 프로세스의 `umask`가 적용된다.

## shell로 blocking과 EOF 관찰

첫 terminal에서 다음 명령을 실행했다.

```bash
cat demo.fifo
```

Writer가 없으므로 `cat`은 FIFO를 read-only로 여는 과정에서 block됐다.

두 번째 terminal에서 다음 명령을 실행했다.

```bash
printf 'hello through FIFO\n' > demo.fifo
```

전체 흐름은 다음과 같다.

```text
cat이 O_RDONLY open에서 writer 대기
→ shell이 demo.fifo를 O_WRONLY로 open
→ 양쪽 open 완료
→ printf의 stdout이 FIFO로 전달
→ cat이 FIFO에서 read
→ writer close
→ cat의 다음 read가 0, EOF
→ cat 종료
```

통신이 끝난 뒤 `cat demo.fifo`를 다시 실행해도 과거 데이터는 나오지 않고 새로운 writer를 기다렸다. FIFO가 데이터를 지속적으로 저장하지 않는다는 것을 확인했다.

## cat과 file descriptor 추상화

Day 2에서 구현한 `mini-cat`과 실제 `cat`의 핵심 흐름은 같다.

```text
입력 FD에서 read()
→ stdout FD 1에 write()
→ read()가 0이면 종료
```

같은 인터페이스로 여러 kernel object를 다룰 수 있다.

| 입력 대상 | 공통 API | 동작 특성 |
|---|---|---|
| Regular file | `open/read/close` | 데이터 유지, file offset 존재 |
| FIFO | `open/read/close` | 상대 프로세스를 기다림, 데이터 비영속 |
| Terminal | `read/write` | 사용자 입력과 terminal 출력 |
| Socket | `read/write/close` | network buffer 사용 |

동일한 `read()`를 호출하더라도 FD가 가리키는 kernel object에 따라 blocking과 EOF의 의미가 다르다.

```text
Regular file의 read() == 0
→ 현재 offset이 파일 끝에 도달

FIFO의 read() == 0
→ 모든 writer가 닫혔고 buffer도 비었음
```

## FIFO open의 blocking

기본 blocking mode에서 FIFO 양쪽의 `open()`은 상대편을 기다린다.

```text
open(path, O_RDONLY)
→ writer가 FIFO를 열 때까지 대기

open(path, O_WRONLY)
→ reader가 FIFO를 열 때까지 대기
```

Reader와 writer 중 어느 쪽을 먼저 실행해도 상대편이 등장하면 양쪽의 `open()`이 완료된다.

```text
Reader 먼저
→ Reader가 open에서 block
→ Writer 실행
→ 양쪽 연결

Writer 먼저
→ Writer가 open에서 block
→ Reader 실행
→ 양쪽 연결
```

## fifo-reader 구현

`fifo-reader`는 명령행 인자로 받은 FIFO를 `O_RDONLY`로 열고 EOF까지 읽는다.

```bash
./fifo-reader <FIFO_PATH>
```

구현한 코드는 다음과 같다.

```c
#include <errno.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
	if (argc != 2) {
		fprintf(stderr, "usage: %s <path>\n", argv[0]);
		return 1;
	}

	printf("wait for writer\n");
	fflush(stdout);

	int fd = open(argv[1], O_RDONLY);
	if (fd == -1) {
		perror("open");
		return 1;
	}
	printf("connected to writer\n");
	fflush(stdout);

	char buffer[128];
	while (1) {
		ssize_t bytes_read = read(fd, buffer, sizeof(buffer)-1);
		if (bytes_read > 0) {
			buffer[bytes_read] = '\0';
			printf("%s", buffer);
			fflush(stdout);
		} else if (bytes_read == 0) {
			// EOF
			printf("\n");
			break;
		} else if (bytes_read == -1 && errno == EINTR) {
			continue;
		} else {
			perror("read");
			close(fd);
			return 1;
		}
	}

	close(fd);
	return 0;
}
```

Reader의 처리 흐름은 다음과 같다.

```text
usage 검사
→ writer 대기 메시지와 fflush
→ O_RDONLY open
→ read 반복
→ 양수이면 text 출력
→ EINTR이면 재시도
→ 0이면 EOF
→ FD close
```

출력 뒤 `fflush(stdout)`을 사용해 `open()`이나 `read()`에서 block되기 전에 상태 메시지가 user에게 보이도록 했다.

처음에는 usage format string에 `%s`를 넣고 대응하는 `argv[0]`을 전달하지 않아 compiler warning이 발생했다. `%s`에 인자가 없으면 undefined behavior이므로 `argv[0]`을 추가했다.

## fifo-writer 구현

`fifo-writer`는 FIFO pathname과 메시지를 받아 `O_WRONLY`로 열고 데이터를 쓴다.

```bash
./fifo-writer <FIFO_PATH> <MESSAGE>
```

구현한 코드는 다음과 같다.

```c
#include <stddef.h>
#include <stdio.h>
#include <errno.h>
#include <fcntl.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
	if (argc != 3) {
		fprintf(stderr, "usage: %s <fifo_path> <message>\n", argv[0]);
		return 1;
	}

	printf("waiting for reader...\n");
	fflush(stdout);

	int fd = open(argv[1], O_WRONLY);
	if (fd == -1) {
		perror("open");
		return 1;
	}
	printf("connected to reader\n");
	fflush(stdout);

	ssize_t w_result = write(fd, argv[2], strlen(argv[2]));
	if (w_result == -1) {
		perror("write");
		close(fd);
		return 1;
	} else if (w_result < (ssize_t)strlen(argv[2])) {
		fprintf(stderr, "partial write\n");
		close(fd);
		return 1;
	}

	close(fd);
	return 0;
}
```

Writer가 FD를 닫으면 열린 writer가 0개가 된다. Reader가 남은 데이터를 모두 읽은 뒤 다음 `read()`에서 0을 받아 종료한다.

## 통합 테스트

Reader를 먼저 실행하고 writer를 나중에 연결했다.

```bash
./fifo-reader demo.fifo
```

```bash
./fifo-writer demo.fifo "hello from writer"
```

두 프로그램 모두 warning 없이 컴파일됐으며 메시지가 정상 전달됐다.

```text
wait for writer
connected to writer
hello from writer
```

Reader와 writer의 종료 상태는 모두 0이었다.

상태 메시지 뒤에서 `fflush()`를 하지 않았을 때 reader를 먼저 실행했어도 writer 출력이 먼저 보이는 현상이 나타났다. Terminal에서는 newline으로 flush될 수 있지만 stdout이 pipe나 파일로 연결되면 block 직전의 출력이 user-space buffer에 남을 수 있다는 것을 확인했다.

## strace로 blocking 시간 관찰

Reader를 다음 옵션으로 추적했다.

```bash
strace -tt -T -yy \
    -e trace=openat,read,write,close \
    ./fifo-reader demo.fifo
```

옵션의 의미는 다음과 같다.

```text
-tt → system call의 시각 표시
-T  → system call 내부에서 소비한 시간 표시
-yy → FD가 가리키는 대상 표시
```

FIFO를 여는 호출은 다음과 같았다.

```text
12:35:53.795922 openat(
    AT_FDCWD</home/kepp/project/linux-system-practice/ipc>,
    "demo.fifo",
    O_RDONLY
) = 3</home/kepp/project/linux-system-practice/ipc/demo.fifo>
    <8.105546>
```

Reader의 `openat()`은 writer가 등장할 때까지 약 8.1초 동안 kernel에서 block됐다. Busy loop로 CPU를 사용하는 것이 아니라 sleeping 상태로 상대편을 기다린다.

`AT_FDCWD`는 상대 경로 `demo.fifo`를 현재 작업 디렉터리 기준으로 해석했다는 뜻이다. `openat()`은 FIFO의 read end를 FD 3으로 반환했다.

Writer가 연결된 뒤 데이터 흐름은 다음처럼 나타났다.

```text
write(1</dev/pts/3<char 136:3>>, "connected to writer\n", 20) = 20
read(3<.../demo.fifo>, "hello", 127) = 5
write(1</dev/pts/3<char 136:3>>, "hello", 5) = 5
read(3<.../demo.fifo>, "", 127) = 0
```

Reader는 최대 127 byte를 요청했지만 준비된 데이터 5 byte를 정상적으로 반환받았다. 이는 short read가 오류가 아님을 다시 보여준다.

Writer가 닫힌 뒤 열린 writer가 0개가 되고 buffer가 비어 다음 `read()`가 즉시 0을 반환했다.

`printf()`와 `fflush()`는 최종적으로 pseudo-terminal `/dev/pts/3`을 가리키는 stdout FD 1에 대한 `write()`로 나타났다.

## 남은 개선 사항

- Reader가 EOF에서 개행만 출력하지 않고 명시적인 EOF 메시지를 출력한다.
- Text가 아닌 임의 binary data는 null 종료와 `%s` 대신 실제 byte 수만큼 stdout에 쓴다.
- Writer에서 `strlen(argv[2])` 결과를 한 번만 계산해 재사용한다.
- Writer의 `w_result`를 `bytes_written`처럼 의미가 분명한 이름으로 변경한다.
- 현재 직접 사용하지 않는 writer의 `<stddef.h>`와 `<errno.h>`를 정리한다.
- Partial write를 감지만 하지 않고 남은 부분부터 재시도한다.
- `close()` 실패 처리 정책을 정한다.
- Production 코드에서는 pathname이 실제 FIFO인지 검증하고 안전한 생성·권한·cleanup 정책을 사용한다.

최종 리뷰 시 `demo.fifo`가 아직 작업 디렉터리에 남아 있었다. 실행 중인 reader와 writer가 없는 것을 확인한 뒤 정확한 파일만 제거해야 한다.

```bash
rm demo.fifo
```

## Linux System Software 직무와의 연결

FIFO 실습을 통해 서로 관련 없는 프로세스가 filesystem namespace를 이용해 IPC endpoint를 찾는 방법을 배웠다. 또한 동일한 `open/read/write/close` 인터페이스가 regular file, FIFO, terminal, socket, device file에 공통으로 적용되지만 각 object의 blocking과 EOF semantics는 다르다는 점을 확인했다.

이 관점은 이후 socket programming과 `/dev/keppdev` character device를 다룰 때 그대로 이어진다.

## 학습 정리

### Learned

- FIFO는 pathname을 가진 named pipe다.
- FIFO의 이름과 inode는 filesystem에 있지만 통신 데이터는 kernel buffer에 존재한다.
- Reader와 writer의 blocking `open()`은 상대편이 나타날 때까지 기다린다.
- 마지막 writer가 닫히고 buffer가 비면 reader의 `read()`가 0을 반환한다.
- 동일한 FD API가 여러 kernel object에 공통으로 적용된다.
- Block 직전의 상태 출력을 확실히 보이게 하려면 stdio buffering을 고려해야 한다.
- `strace -T`로 system call의 blocking 시간을 측정할 수 있다.

### APIs

- `open()`
- `read()`
- `write()`
- `close()`
- `fflush()`

### Commands

- `mkfifo demo.fifo`
- `stat demo.fifo`
- `file demo.fifo`
- `cat demo.fifo`
- `strace -tt -T -yy -e trace=openat,read,write,close`

### Mistakes

- Reader의 usage format에서 `%s`에 대응하는 `argv[0]`을 누락했다.
- User-space 코드에 kernel 내부 errno header를 직접 포함했다.
- Block 직전 메시지에서 stdout buffer를 고려하지 않아 출력 순서가 예상과 달랐다.
- Writer에서 같은 메시지 길이를 반복 계산했다.

### Next

- GDB 기본 명령으로 source-level debugging을 시작한다.
- 이후 shared memory와 synchronization IPC로 확장한다.

