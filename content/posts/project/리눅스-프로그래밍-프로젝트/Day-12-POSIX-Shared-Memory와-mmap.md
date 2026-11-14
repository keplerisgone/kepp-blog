---
title: "Day 12: POSIX Shared Memory와 mmap"
date: "2026-08-25T14:22:01+09:00"
lastmod: "2026-08-25T14:22:01+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-12-posix-shared-memory와-mmap/"
weight: 12
ShowToc: true
---

이번 실습의 목표는 POSIX shared memory object를 생성하고, 서로 다른 두 프로세스가 같은 backing memory를 각자의 virtual address space에 mapping해 데이터를 공유하도록 만드는 것이다.

Writer는 `shm_open()`, `ftruncate()`, `mmap()`으로 shared object를 만들고 문자열을 기록했다. Reader는 같은 이름의 object를 read-only로 mapping해 문자열을 읽은 뒤 `shm_unlink()`로 이름을 제거했다.

```text
Writer virtual address ─┐
                        ├─ Shared backing pages
Reader virtual address ─┘
```

## Pipe, FIFO와 Shared Memory

Pipe와 FIFO는 `write()`와 `read()`를 통해 kernel pipe buffer로 데이터를 전달한다.

```text
Writer user buffer
→ write()
→ Kernel pipe buffer
→ read()
→ Reader user buffer
```

Shared memory는 동일한 backing pages를 여러 프로세스의 virtual address space에 mapping한다.

```text
Writer mapping
      │
      ├─ 같은 backing pages
      │
Reader mapping
```

Mapping을 만든 뒤에는 일반 pointer와 memory 연산으로 데이터를 읽고 쓸 수 있다. 매번 `read()`와 `write()`를 호출해 pipe buffer를 거치지 않는다.

```text
Pipe/FIFO
→ kernel이 제공하는 stream과 EOF semantics
→ read/write가 통신 동작

Shared Memory
→ memory 자체를 공유
→ load/store가 데이터 접근
→ synchronization은 프로그램이 별도로 설계
```

## Shared Memory와 synchronization

`MAP_SHARED`는 같은 memory의 변경 내용을 다른 mapping에서도 볼 수 있게 하지만 접근 순서를 조정하지 않는다.

Writer가 여러 field를 수정하는 중 Reader가 동시에 읽으면 일부만 새 값인 상태를 관찰할 수 있다.

```text
struct metrics
├─ cpu     새 값
├─ memory  이전 값
└─ count   이전 값
```

실제 동시 접근에는 다음과 같은 synchronization이 필요하다.

```text
semaphore
process-shared mutex
atomic
event notification
```

이번 실습에서는 synchronization 문제를 분리하기 위해 Writer가 모든 작업을 마친 뒤 Reader를 실행했다.

```text
Writer 작성 및 종료
→ Reader open 및 읽기
```

## POSIX Shared Memory 이름

다음 이름을 사용했다.

```text
/kepp_shm_demo
```

POSIX shared memory 이름은 `/`로 시작한다. Portable한 이름에는 처음 이외의 `/`를 넣지 않는다.

Linux에서는 일반적으로 `/dev/shm`에 mount된 tmpfs를 이용해 구현되며 다음 경로에서 관찰할 수 있다.

```text
/dev/shm/kepp_shm_demo
```

Portable한 프로그램은 `/dev/shm` 경로를 직접 열지 않고 다음 API를 사용한다.

```c
shm_open("/kepp_shm_demo", ...);
```

## shm_open()

Writer는 기존 object를 실수로 덮어쓰지 않도록 다음 flag를 사용했다.

```text
O_CREAT | O_EXCL | O_RDWR
```

```text
O_CREAT → object가 없으면 생성
O_EXCL  → 이미 존재하면 EEXIST로 실패
O_RDWR  → read/write 가능한 FD
```

권한은 소유자만 접근할 수 있는 `0600`으로 설정했다.

Reader는 기존 object를 생성하지 않고 read-only로 열었다.

```text
O_RDONLY
```

`shm_open()` 반환값은 다음과 같다.

```text
성공 → file descriptor
실패 → -1, errno 설정
```

## ftruncate()

새 POSIX shared memory object의 초기 크기는 0이다.

```text
shm_open 직후 → size 0
ftruncate      → size 4096
```

Writer는 다음 크기로 object를 확장했다.

```text
4096 bytes
```

크기가 0이거나 mapping보다 작은 object의 범위를 실제로 접근하면 mapping 자체가 존재하더라도 backing page가 없어 `SIGBUS`가 발생할 수 있다.

`ftruncate()`로 확장된 새 영역은 0으로 초기화된다.

## mmap()

Writer mapping:

```text
length  → 4096
protect → PROT_READ | PROT_WRITE
flags   → MAP_SHARED
offset  → 0
```

Reader mapping:

```text
length  → 4096
protect → PROT_READ
flags   → MAP_SHARED
offset  → 0
```

`mmap()`의 실패 반환값은 `NULL`이 아니라 `MAP_FAILED`다.

```c
if (mapping == MAP_FAILED) {
    /* error */
}
```

Reader mapping은 `PROT_READ`만 사용하므로 해당 주소에 쓰기를 시도하면 memory protection 위반으로 `SIGSEGV`가 발생할 수 있다.

## MAP_SHARED와 MAP_PRIVATE

`MAP_SHARED` mapping의 변경은 같은 object 영역을 공유 mapping한 다른 프로세스에서 관찰할 수 있다.

```text
Writer가 mapping에 기록
→ shared backing pages 변경
→ Reader mapping에서 같은 내용 관찰
```

`MAP_PRIVATE`는 copy-on-write mapping이다.

```text
Process가 private mapping 변경
→ 해당 process의 private copy
→ 다른 mapping에 변경 내용이 보이지 않음
```

Process 간 데이터 공유에는 `MAP_SHARED`를 사용했다.

## Virtual address와 backing pages

Writer와 Reader의 mapping 주소는 서로 달랐다.

```text
Writer: 0x7bbcced70000
Reader: 0x7ef2afcc5000
```

Virtual address는 프로세스마다 독립적이므로 같은 값일 필요가 없다. 두 mapping은 같은 named object의 backing pages를 가리킨다.

```text
Writer VA 0x7bb... ─┐
                     ├─ /kepp_shm_demo backing pages
Reader VA 0x7ef... ─┘
```

따라서 Writer와 Reader는 서로 다른 pointer 값으로 같은 문자열을 관찰할 수 있었다.

## FD, mapping, 이름의 lifetime

Shared memory에서는 세 종류의 resource를 구분해야 한다.

```text
FD
→ shm_open()이 반환
→ close()로 제거

Mapping
→ mmap()이 생성
→ munmap()으로 제거

Named object
→ shm_open(O_CREAT)이 생성
→ shm_unlink()로 이름 제거
```

`mmap()`에 성공하면 FD를 닫아도 mapping은 유지된다.

```text
shm_open() → FD 획득
mmap()     → 별도의 mapping 참조 생성
close(fd)  → FD만 제거
mapping    → 계속 유효
```

`munmap()`은 현재 프로세스의 virtual address mapping만 제거한다. Shared object 이름이나 다른 프로세스의 mapping을 제거하지 않는다.

`shm_unlink()`은 이름을 제거한다. 호출 즉시 모든 mapping을 무효화하고 memory를 강제로 해제하는 것은 아니다. 열린 FD와 mapping 같은 마지막 참조가 사라질 때 object가 실제로 파괴된다.

```text
shm_unlink
→ 새 프로세스가 이름으로 open할 수 없음
→ 기존 mapping은 계속 사용 가능
→ 마지막 참조 제거 시 실제 object 파괴
```

일반 파일의 `unlink()` lifetime과 비슷하다.

## shm-writer 구현

Writer의 전체 흐름은 다음과 같다.

```text
메시지 길이 검사
→ shm_open으로 새 object 생성
→ ftruncate로 4096 bytes 설정
→ read/write MAP_SHARED mapping
→ FD close
→ null terminator까지 memcpy
→ munmap
→ Reader를 위해 이름 유지
```

구현한 코드는 다음과 같다.

```c
#include <stddef.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

#define SHM_SIZE 4096
static const char shm_name[] = "/kepp_shm_demo";

int main(int argc, char *argv[]) {
	if (argc != 2) {
		fprintf(stderr, "usage: %s <message>\n", argv[0]);
		return 1;
	}

	size_t message_len = strlen(argv[1]);
	if (message_len >= SHM_SIZE) {
		fprintf(stderr, "message is too long\n");
		return 1;
	}

	int fd = shm_open(shm_name, O_CREAT | O_EXCL | O_RDWR, 0600);
	if (fd == -1) {
		perror("shm_open");
		return 1;
	}

	if (ftruncate(fd, SHM_SIZE) == -1) {
		perror("ftruncate");
		close(fd);
		shm_unlink(shm_name);
		return 1;
	}

	char *mmap_result = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
	if (mmap_result == MAP_FAILED) {
		perror("mmap");
		close(fd);
		shm_unlink(shm_name);
		return 1;
	}

	if (close(fd) == -1) {
		perror("close");
		munmap(mmap_result, SHM_SIZE);
		shm_unlink(shm_name);
		return 1;
	}

	memcpy(mmap_result, argv[1], message_len + 1);
	if (munmap(mmap_result, SHM_SIZE) == -1) {
		perror("munmap");
		shm_unlink(shm_name);
		return 1;
	}

	return 0;
}
```

메시지 길이가 4096 이상이면 마지막 null terminator를 넣을 공간이 없으므로 거부했다.

```text
최대 문자 수 → 4095
마지막 byte  → '\0'
```

`memcpy()`에는 `message_len + 1`을 전달해 null terminator까지 저장했다.

## shm-reader 구현

Reader의 전체 흐름은 다음과 같다.

```text
기존 object를 read-only로 open
→ read-only MAP_SHARED mapping
→ FD close
→ mapping의 문자열 출력
→ munmap
→ shm_unlink
```

구현한 코드는 다음과 같다.

```c
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

#define SHM_SIZE 4096
static const char shm_name[] = "/kepp_shm_demo";

int main(void) {
	int fd = shm_open(shm_name, O_RDONLY, 0);
	if (fd == -1) {
		perror("shm_open");
		return 1;
	}

	char *mapping = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);
	if (mapping == MAP_FAILED) {
		perror("mmap");
		close(fd);
		shm_unlink(shm_name);
		return 1;
	}

	if (close(fd) == -1) {
		perror("close");
		munmap(mapping, SHM_SIZE);
		shm_unlink(shm_name);
		return 1;
	}

	printf("Reader: %s\n", mapping);
	if (munmap(mapping, SHM_SIZE) == -1) {
		perror("munmap");
		shm_unlink(shm_name);
		return 1;
	}

	if (shm_unlink(shm_name) == -1) {
		perror("shm_unlink");
		return 1;
	}

	return 0;
}
```

Writer가 mapping 안에 올바른 null-terminated 문자열을 저장한다는 두 프로그램 사이의 contract를 사용했다. Production 코드에서는 object 크기와 null terminator가 mapping 범위 안에 존재하는지 검증해야 한다.

## /dev/shm 관찰

Writer 실행 후 Linux의 object를 확인했다.

```text
-rw------- 1 kepp kepp 4096 ... /dev/shm/kepp_shm_demo
type=regular file size=4096 mode=600
```

Linux는 POSIX shared memory를 일반적으로 `/dev/shm`에 mount된 tmpfs의 regular file 형태로 구현한다. 이는 Linux 구현 세부사항이며 portable interface는 `shm_open()`이다.

Object 내용을 `od`로 확인했다.

```text
0000000   h   e   l   l   o       f   r   o   m       s   h   a   r   e
0000020   d       m   e   m   o   r   y  \0  \0  \0  \0  \0  \0  \0  \0
0000040  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
*
0010000
```

`od`의 기본 offset은 8진수다.

```text
0000020 octal → 16 decimal
0010000 octal → 4096 decimal
```

문자열과 null terminator 뒤의 나머지 영역이 0으로 채워져 있었다. `*`는 동일한 0의 반복 줄을 `od`가 압축했다는 뜻이다.

Reader 실행 후 object가 `/dev/shm`에서 사라졌으며 다시 Reader를 실행하면 `shm_open()`이 `ENOENT`로 실패하는 lifetime도 확인했다.

## strace로 Writer 관찰

다음 명령을 사용했다.

```bash
strace -yy \
    -e trace=openat,ftruncate,mmap,munmap,close,unlink \
    ./shm-writer "hello from strace"
```

Dynamic linker의 libc mapping 뒤에 shared memory 관련 호출이 나타났다.

```text
openat(...,
    "/dev/shm/kepp_shm_demo",
    O_RDWR|O_CREAT|O_EXCL|O_NOFOLLOW|O_CLOEXEC,
    0600
) = 3</dev/shm/kepp_shm_demo>

ftruncate(3</dev/shm/kepp_shm_demo>, 4096) = 0

mmap(NULL, 4096,
    PROT_READ|PROT_WRITE,
    MAP_SHARED,
    3</dev/shm/kepp_shm_demo>, 0
) = 0x7bbcced70000

close(3</dev/shm/kepp_shm_demo>) = 0
munmap(0x7bbcced70000, 4096) = 0
```

소스에서는 `shm_open()`을 호출했지만 Linux libc가 `/dev/shm`에 대한 `openat()`으로 구현했다. `O_NOFOLLOW`와 `O_CLOEXEC`은 libc가 안전성을 위해 추가했다.

`memcpy()`는 system call이 아니라 CPU가 user-space mapping에 수행하는 일반 memory load/store다. `strace`는 system call과 signal을 추적하므로 `memcpy()` 자체는 출력되지 않았다. Page fault가 발생하면 kernel이 개입할 수 있지만 page fault도 user가 호출한 system call이 아니므로 이 trace에 나타나지 않는다.

## strace로 Reader 관찰

Reader trace의 핵심은 다음과 같았다.

```text
openat(...,
    "/dev/shm/kepp_shm_demo",
    O_RDONLY|O_NOFOLLOW|O_CLOEXEC
) = 3</dev/shm/kepp_shm_demo>

mmap(NULL, 4096,
    PROT_READ,
    MAP_SHARED,
    3</dev/shm/kepp_shm_demo>, 0
) = 0x7ef2afcc5000

close(3</dev/shm/kepp_shm_demo>) = 0
Reader: hello from strace
munmap(0x7ef2afcc5000, 4096) = 0
unlink("/dev/shm/kepp_shm_demo") = 0
```

Writer와 Reader의 virtual address는 달랐지만 같은 backing object를 공유했다. 소스의 `shm_unlink()`는 Linux system call 수준에서 `/dev/shm/kepp_shm_demo`에 대한 `unlink()`로 나타났다.

Reader trace filter에 `write`를 포함하지 않았으므로 `printf()`의 최종 stdout `write()`는 trace에 표시되지 않았다.

## 구현 중 확인한 사항

### mmap 실패 반환값

`mmap()`은 실패할 때 `NULL`이 아니라 `MAP_FAILED`를 반환한다. 정상 mapping 주소가 0일 가능성과 오류 표현을 구분하는 API contract를 따라야 한다.

### Kernel 내부 header 사용 금지

User-space에서는 public POSIX/Linux API header를 사용하고 `asm-generic` 같은 kernel 내부 계층 header를 직접 include하지 않는다.

### close 실패 시 재시도 금지

Linux에서 `close()`가 오류를 반환해도 FD가 이미 해제됐을 수 있으므로 같은 번호를 무조건 다시 닫지 않았다. 오류를 출력하고 다른 resource를 best-effort로 정리했다.

### Cleanup ownership

이번 실습에서는 Reader가 최종 cleanup 담당자라는 단순한 lifecycle contract를 사용했다. 여러 reader와 writer가 존재하는 실제 시스템에서는 어느 프로세스가 언제 `shm_unlink()`할지 별도 정책이 필요하다.

## 남은 개선 사항

- Writer의 `mmap_result`를 `mapping` 또는 `shared_memory`처럼 역할 중심 이름으로 변경한다.
- Shared memory name과 size를 중복 정의하지 않고 공통 header로 공유한다.
- Reader에서 `fstat()`으로 실제 object 크기가 contract와 일치하는지 검사한다.
- Reader에서 null terminator가 4096 bytes 안에 존재하는지 제한적으로 검사한다.
- Shared string 대신 version과 length를 가진 명시적인 struct layout을 설계한다.
- 동시에 실행되는 Writer와 Reader 사이에 semaphore 또는 process-shared synchronization을 추가한다.
- Cleanup 함수 자체의 오류를 기록하는 공통 정책을 정한다.

## Linux System Software 직무와의 연결

Shared memory는 큰 데이터를 프로세스 사이에서 공유하거나 process worker들이 공통 상태를 볼 때 사용할 수 있다. 높은 성능을 얻는 대신 data layout, lifetime, synchronization, memory visibility를 application이 직접 관리해야 한다.

`sysmon-agent`의 thread들은 한 process의 address space를 기본적으로 공유하지만 mutex 없이 동시에 metrics를 접근하면 race condition이 생긴다. 이번 shared memory 실습은 이후 pthread와 mutex에서 같은 synchronization 문제를 이해하는 기반이 된다.

## 학습 정리

### Learned

- POSIX shared memory object는 `shm_open()`으로 이름 기반 접근한다.
- 새 object의 크기는 0이므로 접근 전에 `ftruncate()`가 필요하다.
- `mmap()` 실패는 `MAP_FAILED`로 검사한다.
- `MAP_SHARED`는 서로 다른 mapping에서 같은 backing pages의 변경을 관찰하게 한다.
- Virtual address가 달라도 underlying object는 공유할 수 있다.
- FD, mapping, object 이름은 서로 다른 lifetime을 가진다.
- `munmap()`과 `shm_unlink()`의 제거 대상은 서로 다르다.
- Shared memory는 데이터를 공유하지만 synchronization을 자동 제공하지 않는다.

### APIs

- `shm_open()`
- `ftruncate()`
- `mmap()`
- `memcpy()`
- `close()`
- `munmap()`
- `shm_unlink()`

### Commands

- `man 3 shm_open`
- `man 2 ftruncate`
- `man 2 mmap`
- `ls -l /dev/shm`
- `stat /dev/shm/kepp_shm_demo`
- `od -c /dev/shm/kepp_shm_demo`
- `strace -yy -e trace=openat,ftruncate,mmap,munmap,close,unlink`

### Mistakes

- `MAP_SHARED`의 영어 설명에서 virtual address와 backing pages의 관계를 처음에 이해하기 어려웠다.
- `shm_unlink()`가 호출 즉시 모든 mapping과 memory를 강제로 해제한다고 생각할 수 있었다.
- Reader의 사용하지 않는 `argc`, `argv`로 compiler warning이 발생했다.
- Reader에서 mapping 성공 후 `close()` 반환값 검사를 처음에 빠뜨렸다.

### Next

- pthread로 한 process 안에 여러 실행 흐름을 만든다.
- mutex로 shared state에 대한 race condition을 방지한다.
- GDB의 `info threads`와 thread별 backtrace를 사용한다.

