---
title: "Day 3: mini-cp 구현하기"
date: "2026-08-04T16:16:27+09:00"
lastmod: "2026-08-04T16:16:27+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-3-mini-cp-구현하기/"
weight: 3
ShowToc: true
---

이번 실습의 목표는 Linux 파일 입출력 시스템 콜을 사용해 하나의 파일을 다른 파일로 복사하는 간단한 `mini-cp` 프로그램을 구현하는 것이다.

Day 2의 mini-cat에서는 파일에서 읽은 데이터를 표준 출력으로 보냈다. 이번 mini-cp에서는 원본 파일과 대상 파일을 각각 열고, 두 개의 file descriptor를 이용해 데이터를 복사한다.

프로그램의 실행 형식은 다음과 같다.

```bash
./mini-cp SOURCE DESTINATION
```

예를 들어 다음 명령은 `source.txt`의 내용을 `destination.txt`로 복사한다.

```bash
./mini-cp source.txt destination.txt
```

실제 `cp` 명령어는 디렉터리 복사, 권한 보존, 심볼릭 링크 처리 등 다양한 기능을 제공한다. 이번 mini-cp에서는 범위를 단순하게 제한했다.

- 하나의 일반 파일만 복사한다.
- 옵션은 처리하지 않는다.
- 디렉터리는 복사하지 않는다.
- 대상 파일이 존재하면 기존 내용을 비운다.
- 파일 권한과 소유권을 그대로 보존하지 않는다.
- partial write 재시도는 TODO로 남긴다. (어렵다)

## 기본 이론

### mini-cat과 mini-cp의 차이

Day 2에서 구현한 mini-cat의 데이터 흐름은 다음과 같았다.

```text
입력 파일 FD
→ read()
→ 메모리 버퍼
→ write()
→ 표준 출력 FD 1
```

이번 mini-cp에서는 `write()`의 대상이 표준 출력이 아니라 대상 파일이다.

```text
원본 파일 FD
→ read()
→ 메모리 버퍼
→ write()
→ 대상 파일 FD
```

두 프로그램 모두 `read()`와 `write()`를 사용하지만, `write()`에 전달하는 FD가 다르다.

mini-cat은 미리 열려 있는 표준 출력 FD인 `STDOUT_FILENO`를 사용한다. mini-cp는 `open()`으로 대상 파일을 열고 반환받은 FD를 사용한다.

### open(), openat(), creat()

`open()`은 파일을 열고 성공하면 음이 아닌 file descriptor를 반환한다.

`openat()`은 기본적으로 `open()`과 비슷하지만, 특정 디렉터리 FD를 기준으로 상대 경로를 처리할 수 있다.

`creat()`은 파일을 생성하거나 기존 파일을 비우는 용도로 만들어진 오래된 편의 함수다. 다음 두 호출은 개념적으로 같은 동작을 한다.

```text
creat(path, mode)

open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)
```

이번 구현에서는 더 다양한 플래그를 조합할 수 있는 `open()`을 사용했다.

소스 코드에서 `open()`을 호출해도 `strace`에서는 `openat()` 시스템 콜로 보일 수 있다. Linux의 C 라이브러리가 `open()`을 내부적으로 `openat()`을 이용해 구현할 수 있기 때문이다.

### 원본 파일 열기

원본 파일은 데이터를 읽기만 하므로 다음 플래그를 사용한다.

```text
O_RDONLY
```

`open()`의 반환값은 다음과 같다.

- 성공: 음이 아닌 file descriptor
- 실패: -1

원본 파일 열기에 실패하면 유효한 FD가 만들어지지 않는다. 따라서 반환값이 `-1`일 때 `close()`를 호출하면 안 된다.

### 대상 파일 열기

대상 파일에는 다음 플래그를 조합했다.

```text
O_WRONLY | O_CREAT | O_TRUNC
```

각 플래그의 의미는 다음과 같다.

- `O_WRONLY`: 파일을 쓰기 전용으로 연다.
- `O_CREAT`: 파일이 존재하지 않으면 새로 만든다.
- `O_TRUNC`: 파일이 이미 존재하면 길이를 0으로 만든다.

`O_TRUNC`는 `open()`이 성공하는 순간 기존 내용을 제거한다. 따라서 원본과 대상에 같은 파일을 지정하면 원본 내용도 사라질 수 있다.

`O_CREAT`를 사용할 때는 세 번째 인자로 새 파일의 권한을 전달해야 한다. 이 형태는 `chmod`로 권한을 부여할 때와 비슷하다.

이번 구현에서는 다음 값을 사용했다.

```text
0644
```

각 권한의 의미는 다음과 같다.

- 소유자: 읽기 및 쓰기
- 그룹: 읽기
- 기타 사용자: 읽기

다만 실제 생성되는 파일 권한은 프로세스의 `umask`에 영향을 받는다.

개념적으로 최종 권한은 다음과 같이 결정된다.

```text
요청한 mode에서 umask로 제한된 권한을 제거
```

### File descriptor 두 개 관리하기

원본과 대상을 차례대로 열면 프로세스에는 두 개의 FD가 생긴다.

일반적으로 0, 1, 2번은 표준 입력, 표준 출력, 표준 에러에 사용되고 있으므로 이번 `strace`에서는 다음 번호가 할당됐다.

```text
FD 3 → 원본 파일
FD 4 → 대상 파일
```

FD 번호가 항상 3과 4인 것은 아니다. 커널은 일반적으로 현재 사용하지 않는 가장 작은 FD 번호를 할당한다.

두 파일을 모두 열었으므로 프로그램을 종료하기 전에 두 FD를 모두 닫아야 한다.

### read()

`read()`는 원본 FD에서 데이터를 읽어 메모리 버퍼에 저장한다.

반환값은 다음과 같이 구분한다.

- 양수: 실제로 읽은 바이트 수
- 0: EOF
- -1: 오류

이번 구현에서는 4096바이트 크기의 고정 버퍼를 사용했다.

```text
BUF_SIZE = 4096
```

`read()`가 항상 4096바이트를 반환하는 것은 아니다. 파일의 남은 크기나 시스템 상태에 따라 더 적은 값을 반환할 수 있다.

마지막 데이터까지 모두 읽은 뒤 다시 `read()`를 호출하면 0이 반환된다. 이는 현재 파일 오프셋이 EOF에 도달했다는 뜻이다.

### write()

`write()`에는 대상 파일 FD, 버퍼 주소, 실제로 읽은 바이트 수를 전달한다.

```text
write(fd_dest, buf, r_result)
```

버퍼 전체 크기인 4096이 아니라 `read()`가 실제로 반환한 `r_result`만큼 기록해야 한다.

예를 들어 마지막 `read()`가 57바이트를 반환했다면 `write()`에도 57바이트를 전달해야 한다. 버퍼 전체를 쓰면 이번에 읽지 않은 영역까지 대상 파일에 기록될 수 있다.

`write()`의 반환값은 다음처럼 처리했다.

- -1: 시스템 콜 실패
- 읽은 크기보다 작은 양수: partial write
- 읽은 크기와 같은 값: 이번 버퍼 기록 완료

현재 구현은 partial write를 감지하면 오류 메시지를 출력하고 실패 상태로 종료한다. 남은 데이터를 이어서 기록하는 처리는 TODO로 남겼다. (아무리 생각해봐도 어렵다)

### perror()와 errno

시스템 콜이 실패하면 일반적으로 `errno`에 실패 원인이 기록된다.

`perror()`는 사용자가 지정한 문맥 문자열과 현재 `errno`의 설명을 함께 표준 에러로 출력한다.

예를 들어 존재하지 않는 원본 파일을 열면 다음과 같은 메시지가 표시된다.

```text
mini-cp: open source: No such file or directory
```

시스템 콜이 `-1`을 반환했을 때는 `perror()`를 사용했다.

하지만 partial write처럼 `write()`가 양수를 반환한 경우에는 `errno`가 유효하다는 보장이 없다. 따라서 이 경우에는 `fprintf(stderr, ...)`를 사용해 실제 기록한 바이트 수를 직접 출력했다.

### FILE *와 file descriptor

처음 오류 메시지를 작성하면서 다음과 같이 `fprintf()`에 정수 2를 전달했다.

```c
fprintf(2, "error message");
```

하지만 `fprintf()`의 첫 번째 인자는 정수 FD가 아니라 `FILE *` 스트림이어야 한다.

표준 에러를 나타내는 두 표현은 다음처럼 서로 타입이 다르다.

```text
stderr         → FILE *
STDERR_FILENO  → 정수 FD, 값은 2
```

따라서 `fprintf()`를 사용할 때는 `stderr`를 전달해야 한다.

```c
fprintf(stderr, "error message");
```

정수 2를 전달하면 컴파일러는 정수를 포인터로 변환한다는 경고를 출력한다. 실제 실행에서는 유효하지 않은 주소를 `FILE` 객체처럼 접근하면서 segmentation fault가 발생했다.

이 경험을 통해 컴파일 경고가 단순한 스타일 문제가 아니라 실제 메모리 오류나 undefined behavior를 나타낼 수 있다는 것을 확인했다.

## mini-cp 구현하기

구현한 전체 코드는 다음과 같다.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#define BUF_SIZE 4096

int main(int argc, char* argv[]) {
if (argc != 3) {
	fprintf(stderr, "usage: %s <SOURCE> <DESTINATION>\n", argv[0]);
	return 1;
}

int fd_source = open(argv[1], O_RDONLY);
if (fd_source == -1) {
	perror("mini-cp: open source");
	return 1;
}

int fd_dest = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
if (fd_dest == -1) {
	perror("mini-cp: open destination");
	if (close(fd_source) == -1) {
		perror("mini-cp: close source");
	}
	return 1;
}

int result = 0;
char buf[BUF_SIZE];
ssize_t r_result = read(fd_source, buf, BUF_SIZE);

while (r_result > 0) {
	ssize_t w_result = write(fd_dest, buf, r_result);
	if (w_result == -1) {
		perror("mini-cp: write destination");
		result = 1;
		break;
	}

	/* TODO: Retry write() until all bytes have been written. */
	if (w_result < r_result) {
		fprintf(
			stderr,
			"mini-cp: partial write: wrote %zd of %zd bytes\n",
			w_result,
			r_result
		);
		result = 1;
		break;
	}

	r_result = read(fd_source, buf, BUF_SIZE);
}

if (r_result == -1) {
	perror("mini-cp: read source");
	result = 1;
}

if (close(fd_dest) == -1) {
	perror("mini-cp: close destination");
	result = 1;
}

if (close(fd_source) != 0) {
	perror("mini-cp: close source");
	result = 1;
}

if (result == 0) {
	printf("good!: %s, %s\n", argv[1], argv[2]);
}

return result;
}
```

## 프로그램의 전체 흐름

구현 전에 프로그램의 처리 순서를 다음과 같이 정리했다.

- 명령행 인자 개수 검사
- 원본 파일을 읽기 전용으로 열기
- 대상 파일을 쓰기 전용으로 열거나 생성하기
- 원본에서 고정 크기만큼 읽기
- 실제로 읽은 바이트 수만큼 대상에 쓰기
- EOF까지 읽기와 쓰기 반복
- 대상 FD 닫기
- 원본 FD 닫기
- 전체 결과에 맞는 종료 상태 반환

데이터 이동만 표현하면 다음과 같다.

```text
원본 파일 FD
→ read()
→ 메모리 버퍼
→ write()
→ 대상 파일 FD
```

## 명령행 인자 검사

프로그램명 외에 원본과 대상 경로가 필요하므로 `argc`는 3이어야 한다.

인자 개수가 맞지 않으면 사용법을 표준 에러에 출력하고 실패 상태로 종료한다.

```text
usage: ./mini-cp <SOURCE> <DESTINATION>
```

프로그램 이름을 직접 `./mini-cp`로 고정하지 않고 `argv[0]`을 사용했다. 이렇게 하면 실행 경로나 프로그램 이름이 달라져도 실제 실행 이름이 오류 메시지에 표시된다.

## 실패 경로와 자원 정리

오류 처리에서 중요한 것은 실패한 시점까지 어떤 자원을 성공적으로 확보했는지 확인하는 것이다.

원본 파일 열기에 실패한 경우에는 열린 FD가 없으므로 바로 종료한다.

```text
원본 open 실패
→ 닫을 FD 없음
→ 실패 반환
```

원본 파일을 연 뒤 대상 파일 열기에 실패하면 원본 FD는 이미 유효한 상태다. 따라서 원본 FD를 닫은 뒤 종료해야 한다.

```text
원본 open 성공
→ 대상 open 실패
→ 원본 FD close
→ 실패 반환
```

두 파일을 모두 연 뒤 읽기나 쓰기가 실패한 경우에는 즉시 `return`하지 않는다. `result`에 실패 상태를 기록한 뒤 반복문을 빠져나와 두 FD를 모두 닫는다.

```text
I/O 실패
→ result에 실패 기록
→ 반복 종료
→ 대상 FD close
→ 원본 FD close
→ 실패 반환
```

처음에는 대상 FD를 닫은 직후 `return`하여 원본 FD를 닫는 코드에 도달하지 못하는 문제가 있었다.

이를 해결하기 위해 `result` 변수를 사용했다. 첫 번째 `close()`가 실패해도 바로 반환하지 않으므로 나머지 FD의 정리를계속 시도할 수 있다.

프로세스가 종료되면 커널이 열린 FD를 회수하지만, 이를 정상적인 자원 정리 방법으로 의존하면 안 된다. 장시간 실행되는 daemon이나 반복적으로 파일을 여는 프로그램에서는 FD 누수가 실제 장애로 이어질 수 있다.

## 컴파일

다음 명령으로 컴파일했다.

```bash
gcc -Wall -Wextra -g -o mini-cp mini-cp.c
```

각 옵션의 의미는 다음과 같다.

- `-Wall`: 자주 발생하는 주요 경고 활성화
- `-Wextra`: 추가적인 경고 활성화
- `-g`: GDB에서 사용할 디버깅 정보 포함
- `-o mini-cp`: 출력 실행 파일 이름 지정

최종 컴파일 결과는 warning 0, error 0이었다.

## 기능 테스트

다음 경우를 테스트했다.

- 일반 텍스트 파일
- 빈 파일
- 버퍼보다 큰 파일
- 바이너리 파일
- 존재하지 않는 원본 파일
- 생성할 수 없는 대상 경로
- 이미 존재하는 대상 파일

일반 텍스트 파일은 다음과 같이 테스트했다.

```bash
printf 'Linux system programming\n' > source.txt
./mini-cp source.txt destination.txt
cmp source.txt destination.txt
echo $?
```

빈 파일도 복사했다.

```bash
: > empty.txt
./mini-cp empty.txt empty-copy.txt
cmp empty.txt empty-copy.txt
echo $?
```

버퍼보다 큰 바이너리 파일은 다음과 같이 만들었다.

```bash
dd if=/dev/urandom of=large.bin bs=4096 count=3 status=none
./mini-cp large.bin large-copy.bin
cmp large.bin large-copy.bin
echo $?
```

`cmp`는 두 파일의 내용이 같으면 아무것도 출력하지 않고 종료 상태 0을 반환한다.

모든 정상 복사 테스트에서 `cmp`의 종료 상태가 0인 것을 확인했다.

존재하지 않는 원본 파일을 전달하면 `perror()`가 다음과 같이 실제 오류 원인을 출력했다.

```text
mini-cp: open source: No such file or directory
```

잘못된 대상 경로를 전달한 경우에도 프로그램은 오류 메시지를 출력하고 0이 아닌 상태로 종료했다.

## strace로 시스템 콜 확인하기

다음 명령으로 mini-cp가 호출하는 파일 입출력 시스템 콜을 관찰했다.

```bash
strace -e trace=openat,read,write,close \
./mini-cp source.txt destination.txt
```

출력에서 원본 파일에는 FD 3, 대상 파일에는 FD 4가 할당된 것을 확인했다.

데이터 이동은 다음과 같이 나타났다.

```text
read(3, ...)
write(4, ...)
```

마지막 `read()`는 0을 반환했다.

```text
read(3, ..., 4096) = 0
```

이는 원본 파일의 EOF에 도달했다는 뜻이다.

복사가 끝난 뒤에는 대상 FD 4와 원본 FD 3에 대해 `close()`가 호출되는 것도 확인했다.

## 구현 중 발생한 문제

### fprintf()에 FD 전달

`fprintf()`의 첫 번째 인자로 정수 2를 전달해 컴파일 경고와 segmentation fault가 발생했다.

이를 통해 `FILE *`와 정수 FD의 차이를 다시 확인했다.

### 실패한 open()의 반환값 닫기

`open()`이 실패해 `-1`을 반환했는데도 해당 값을 `close()`에 전달했다.

실패한 `open()`은 열린 FD를 만들지 않으므로 닫을 대상이 없다는 것을 확인했다.

### 잘못된 FD에 쓰기

처음에는 `write()`의 대상으로 `fd_dest`가 아니라 읽기 전용인 `fd_source`를 전달했다.

다음과 같은 역할을 구분해야 한다.

```text
fd_source → read()
fd_dest   → write()
```

### 도달할 수 없는 자원 정리 코드

대상 FD를 닫은 직후 `return`하여 원본 FD를 닫는 코드가 실행되지 않는 구조를 만들었다.

종료 상태를 변수에 기록하고 마지막에 한 번만 반환하도록 수정했다.

### 비트 OR와 논리 OR 혼동

두 조건을 연결하면서 논리 OR인 `||` 대신 비트 OR인 `|`를 사용했다.

조건식의 참과 거짓을 결합할 때는 논리 연산자를 사용해야 한다.

### 저장하지 않은 파일 검사

Neovim에서 코드를 수정한 뒤 저장하지 않아 디스크의 이전 버전을 검사했다.

현재 파일을 저장하는 명령과 파일 경로 확인 방법은 다음과 같다.

```vim
:w
:echo expand('%:p')
```

## 남은 개선 사항

현재 구현은 partial write를 감지하지만 남은 데이터를 다시 기록하지 않는다.

완전히 처리하려면 다음 과정이 필요하다.

```text
write()가 실제로 기록한 바이트 수 확인
→ 다음 버퍼 위치로 이동
→ 남은 바이트 수 계산
→ 남은 크기가 0이 될 때까지 write() 반복
```

개념적으로 다음 상태를 관리해야 한다.

```text
다음 버퍼 위치 = buf + 지금까지 기록한 크기
남은 크기 = read()로 읽은 크기 - 지금까지 기록한 크기
```

또한 향후 다음 사항을 개선할 수 있다.

- `EINTR` 발생 시 `read()`와 `write()` 재시도
- 원본과 대상이 같은 파일인지 검사
- 원본 파일 권한 보존
- 정상 실행 시 출력되는 `good!` 메시지 제거
- 명령행 옵션과 여러 파일 처리

이번 실습에서는 두 개의 FD를 이용해 원본 파일의 데이터를 대상 파일로 복사하고, 주요 시스템 콜의 실패와 자원 정리를 처리하는 기본 구현까지 완료했다.

Partial write 재시도와 동일 파일 검사는 향후 개선할 TODO로 남겼다.
