---
title: "Day 4: File Metadata와 inode"
date: "2026-08-06T20:00:16+09:00"
lastmod: "2026-08-06T20:00:16+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-4-file-metadata와-inode/"
weight: 4
ShowToc: true
---

이번 실습의 목표는 `lstat()` 시스템 콜을 사용해 파일의 메타데이터를 조회하는 `file-info` 프로그램을 구현하는 것이다.

Day 2와 Day 3에서는 `open()`, `read()`, `write()`, `close()`를 이용해 파일 내용을 다뤘다. 이번에는 파일 내용이 아니라 커널과 파일시스템이 관리하는 파일의 정보를 조회했다.

프로그램의 실행 형식은 다음과 같다.

```bash
./file-info PATH
```

이번 프로그램에서는 다음 정보를 출력하도록 범위를 제한했다.

- 입력받은 경로
- 파일 종류
- 파일 크기
- inode 번호
- hard link 개수
- 접근 권한

## 기본 이론

### 파일 데이터와 메타데이터

파일에는 실제 내용과 파일을 관리하기 위한 메타데이터가 있다.

```text
파일 데이터
→ read()로 읽는 실제 내용

파일 메타데이터
→ 파일 종류, 크기, 권한, inode, link count 등
```

Day 2와 Day 3에서 사용한 `read()`는 파일 데이터를 읽는다. 이번에 사용한 `lstat()`은 파일의 메타데이터를 조회한다.

### inode

inode는 파일시스템이 파일을 관리하기 위해 사용하는 객체다.

inode에는 일반적으로 다음과 같은 정보가 들어 있다.

- 파일 종류
- 접근 권한
- 파일 크기
- 소유자와 그룹
- hard link 개수
- 접근 및 변경 시각
- 실제 데이터 블록을 찾기 위한 정보

파일 이름 자체가 inode에 저장되는 것은 아니다. 디렉터리가 파일 이름과 inode 번호의 대응 관계를 관리한다.

```text
디렉터리의 파일 이름
→ inode 번호
→ 파일 메타데이터와 실제 데이터
```

inode 번호는 하나의 파일시스템 안에서 파일을 식별한다. 서로 다른 파일시스템에서는 같은 inode 번호가 존재할 수 있으므로, 두 경로가 실제로 같은 파일인지 비교하려면 일반적으로 다음 두 값을 함께 확인해야 한다.

```text
st_dev
st_ino
```

`st_dev`는 파일이 속한 장치를, `st_ino`는 해당 파일시스템 안의 inode 번호를 나타낸다.

### stat(), lstat(), fstat()

파일 메타데이터를 조회하는 대표적인 함수는 다음과 같다.

```text
stat()
lstat()
fstat()
```

`stat()`은 파일 경로를 인자로 받는다. 경로가 심볼릭 링크라면 링크를 따라가 최종 대상의 정보를 반환한다.

`lstat()`도 파일 경로를 인자로 받지만, 마지막 경로가 심볼릭 링크라면 링크를 따라가지 않고 심볼릭 링크 자체의 정보를 반환한다. 이번 프로그램에서는 파일 종류가 심볼릭 링크인지 판별하기 위해 `lstat()`을 사용했다.

`fstat()`은 경로 대신 이미 열린 file descriptor를 인자로 받는다.

```text
open()
→ FD 획득
→ fstat(FD, ...)
```

관련 매뉴얼은 다음과 같이 확인했다.

```bash
man 2 stat
man 3type stat
man 7 inode
```

매뉴얼에서 `stat(3type)`이라고 표시된 문서는 이름이 `stat`이고 section이 `3type`이라는 뜻이다.

`lstat()`은 성공하면 0을 반환한다. 실패하면 -1을 반환하고 `errno`를 설정한다.

```text
성공: 0
실패: -1, errno 설정
```

`errno`가 함수의 두 번째 반환값으로 전달되는 것은 아니다. 실패를 확인한 뒤 `perror()` 등을 이용해 현재 `errno`의 의미를 출력할 수 있다.

### struct stat

`stat()` 계열 함수는 조회 결과를 `struct stat` 구조체에 기록한다.

이번 실습에서는 다음 필드를 사용했다.

```text
st_dev
st_ino
st_mode
st_nlink
st_size
```

각 필드의 의미는 다음과 같다.

- `st_dev`: 파일이 속한 장치 또는 파일시스템 식별 정보
- `st_ino`: inode 번호
- `st_mode`: 파일 종류와 접근 권한
- `st_nlink`: hard link 개수
- `st_size`: 파일 크기

### st_mode와 파일 종류

`st_mode`에는 파일 종류와 접근 권한 비트가 함께 저장된다.

```text
st_mode
= 파일 종류 비트 + 접근 권한 비트
```

따라서 `st_mode` 전체를 `S_IFREG` 같은 값과 바로 비교하면 안 된다.

파일 종류를 검사하는 한 가지 방법은 다음 판별 매크로를 사용하는 것이다.

```c
S_ISREG(sb.st_mode)
S_ISDIR(sb.st_mode)
S_ISLNK(sb.st_mode)
```

이번 구현에서는 `man` 페이지의 예제를 참고해 파일 종류 비트만 마스킹한 뒤 `switch`로 구분했다.

```c
switch (sb.st_mode & S_IFMT) {
    case S_IFREG:
        break;
    case S_IFDIR:
        break;
    case S_IFLNK:
        break;
}
```

`S_IFMT`를 적용하면 접근 권한 비트가 제거되고 파일 종류 비트만 남는다.

```text
S_IFREG → 일반 파일
S_IFDIR → 디렉터리
S_IFLNK → 심볼릭 링크
```

처음에는 위 세 종류만 처리했지만, 다른 파일 종류가 아무 출력 없이 성공하지 않도록 `default`에서 `other`를 출력하도록 추가했다.

### 접근 권한 출력

`st_mode`에서 접근 권한만 출력하려면 다음 마스크를 적용한다.

```text
sb.st_mode & 0777
```

권한은 8진수로 표현하기 위해 `%o` 형식 지정자를 사용했다. 네 자리 형태로 출력하기 위해 `%04o`를 사용했다.

```text
0644
0664
0755
0777
```

### 메타데이터 필드의 타입

`struct stat`의 필드들은 일반적인 `int` 타입이라고 가정하면 안 된다.

```text
st_size  → off_t
st_ino   → ino_t
st_nlink → nlink_t
st_mode  → mode_t
```

잘못된 형식 지정자를 사용하면 컴파일 경고가 발생하거나 undefined behavior가 생길 수 있다.

이번 구현에서는 충분히 큰 표준 정수형으로 변환해 출력했다.

```text
st_size  → intmax_t,  %jd
st_ino   → uintmax_t, %ju
st_nlink → uintmax_t, %ju
```

이를 위해 `<stdint.h>`를 포함했다.

## Hard link와 symbolic link

### Hard link

Hard link는 동일한 inode를 가리키는 또 하나의 파일 이름이다.

```text
original.txt ───┐
                ├──→ 같은 inode → 같은 파일 데이터
hard-link.txt ──┘
```

둘 중 하나가 원본이고 다른 하나가 복사본인 관계가 아니다. 두 이름은 동일한 inode를 가리키는 동등한 이름이다.

Hard link는 다음 명령으로 만들 수 있다.

```bash
ln original.txt hard-link.txt
```

같은 inode를 가리키는 파일 이름이 두 개이므로 link count도 2가 된다. 한쪽 이름을 통해 파일 내용을 수정하면 다른 이름으로 읽어도 같은 변경 내용이 보인다.

하나의 이름을 삭제해도 다른 hard link가 남아 있으면 파일 데이터에 계속 접근할 수 있다. 일반적으로 link count가 0이 되고 해당 파일을 열고 있는 프로세스도 없어야 파일 데이터가 최종적으로 회수된다.

Hard link에는 다음과 같은 제약이 있다.

- 다른 파일시스템을 가로질러 만들 수 없다.
- 일반 사용자는 보통 디렉터리에 hard link를 만들 수 없다.
- 같은 inode를 직접 공유하므로 대상 경로가 사라져 깨지는 구조가 아니다.

### Symbolic link

Symbolic link는 대상 파일의 경로 문자열을 저장하는 별도의 파일이다.

```text
symbolic-link.txt
→ 별도의 inode
→ 내부 내용: "original.txt"
→ 경로를 따라 original.txt 접근
```

다음 명령으로 symbolic link를 만들었다.

```bash
ln -s original.txt symbolic-link.txt
```

심볼릭 링크 자체와 대상 파일은 서로 다른 inode를 가진다.

```text
lstat(symbolic-link.txt)
→ symbolic-link.txt 자체의 inode

stat(symbolic-link.txt)
→ 링크를 따라간 대상의 inode
```

GNU `stat` 명령에서는 다음과 같이 링크 자체와 링크의 대상을 비교했다.

```bash
stat symbolic-link.txt
stat -L symbolic-link.txt
```

기본 `stat` 명령에서는 심볼릭 링크 자체의 정보가 표시됐다. `-L` 옵션을 사용하면 링크를 따라가 `original.txt`의 inode가 표시됐다.

Hard link와 symbolic link의 차이는 다음과 같다.

| 구분 | Hard link | Symbolic link |
|---|---|---|
| inode | 대상과 동일 | 별도의 inode |
| 저장 대상 | 동일 inode에 대한 이름 | 대상 경로 문자열 |
| 대상 이름 삭제 | 다른 hard link로 접근 가능 | 링크가 깨질 수 있음 |
| 다른 파일시스템 | 불가능 | 가능 |
| 디렉터리 링크 | 일반적으로 불가능 | 가능 |
| `lstat()` 결과 | 일반 파일 정보 | 링크 자체 정보 |

## file-info 구현하기

구현한 전체 코드는 다음과 같다.

```c
#include <stdio.h>
#include <sys/stat.h>
#include <stdint.h>

int main(int argc, char* argv[]) {
	if (argc != 2) {
		fprintf(stderr, "usage: %s <PATH>\n", argv[0]);
		return 1;
	}

	struct stat sb;

	if (lstat(argv[1], &sb) == -1) {
		perror("lstat error");
		return 1;
	}

	printf("Path: %s\n", argv[1]);

	switch (sb.st_mode & S_IFMT) {
		case S_IFREG: printf("Type: regular file\n"); break;
		case S_IFDIR: printf("Type: directory\n"); break;
		case S_IFLNK: printf("Type: symbolic link\n"); break;
		default: printf("Type: other\n");
	}

	printf("Size: %jd\n", (intmax_t) sb.st_size);
	printf("Inode: %ju\n", (uintmax_t) sb.st_ino);
	printf("Links: %ju\n", (uintmax_t) sb.st_nlink);
	printf("Mode: %04o\n", (unsigned int) sb.st_mode & 0777);

	return 0;
}
```

## 프로그램의 전체 흐름

프로그램의 처리 순서는 다음과 같다.

```text
명령행 인자 검사
→ struct stat 변수 선언
→ lstat()으로 경로 조회
→ 파일 종류 판별
→ 크기, inode, link count, 권한 출력
→ 종료 상태 반환
```

이번 프로그램에서는 파일이나 FD를 직접 열지 않았다. `lstat()`이 메타데이터를 구조체에 복사하므로 프로그램에서 `close()`할 자원은 없다.

### 명령행 인자 검사

프로그램은 하나의 파일 경로만 입력받으므로 프로그램 이름을 포함한 `argc`는 2여야 한다.

인자 개수가 맞지 않으면 사용법을 표준 에러에 출력하고 실패 상태로 종료한다.

```text
usage: ./file-info <PATH>
```

### lstat() 호출과 오류 처리

`struct stat` 타입의 변수를 선언하고 `lstat()`의 두 번째 인자로 주소를 전달했다.

```text
경로
→ lstat()
→ struct stat에 메타데이터 저장
```

존재하지 않는 경로에서는 `lstat()`이 -1을 반환하고 `errno`를 설정한다. `perror()`로 실제 오류 원인을 출력하고 실패 상태 1로 종료하도록 했다.

```text
lstat error: No such file or directory
```

### 파일 종류와 메타데이터 출력

`st_mode`에 `S_IFMT` 마스크를 적용해 파일 종류 비트만 남기고 `switch`로 파일 종류를 출력했다.

파일 크기, inode, hard link 개수는 각 필드 타입을 고려해 `intmax_t` 또는 `uintmax_t`로 변환했다. 권한은 `0777` 마스크를 적용한 뒤 8진수로 출력했다.

일반 파일에서는 다음과 같은 결과를 얻었다.

```text
Path: metadata-test/original.txt
Type: regular file
Size: 16
Inode: 22697009
Links: 2
Mode: 0664
```

심볼릭 링크에서는 다음과 같은 차이가 나타났다.

```text
Type: symbolic link
Size: 12
Mode: 0777
```

심볼릭 링크의 크기 12는 링크에 저장된 `original.txt` 경로 문자열의 바이트 수다.

디렉터리의 `st_size`는 내부에 있는 파일 크기의 합이 아니다. 디렉터리 엔트리를 저장하기 위해 파일시스템이 사용하는 크기이며 파일시스템 구현에 따라 달라질 수 있다.

## 컴파일

다음 명령으로 컴파일했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -o file-info file-info.c
```

각 옵션의 의미는 다음과 같다.

- `-Wall`: 자주 발생하는 주요 경고 활성화
- `-Wextra`: 추가적인 경고 활성화
- `-Wpedantic`: C 표준을 엄격하게 벗어나는 사용 경고
- `-g`: GDB에서 사용할 디버깅 정보 포함
- `-o file-info`: 출력 실행 파일 이름 지정

최종 컴파일 결과는 warning 0, error 0이었다.

## 기능 테스트

다음 종류의 경로를 테스트했다.

- 일반 파일
- 디렉터리
- hard link
- symbolic link
- 존재하지 않는 경로

```bash
./file-info metadata-test/original.txt
./file-info metadata-test
./file-info metadata-test/symbolic-link.txt
./file-info metadata-test/nonexistent.txt
```

존재하지 않는 경로를 전달하면 오류 메시지와 실패 종료 상태 1이 반환되는 것을 확인했다.

직접 구현한 결과는 다음 명령의 결과와 비교했다.

```bash
stat metadata-test/original.txt
ls -li metadata-test/original.txt
```

파일 종류, 크기, inode, link count, 권한이 모두 일치하는 것을 확인했다.

## cp -r로 hard link 관계가 사라진 사례

처음 생성한 `original.txt`와 `hard-link.txt`는 같은 inode를 가리켰고 link count는 2였다.

이후 실습 디렉터리를 다음 명령으로 다른 위치에 복사했다.

```bash
cp -r metadata-test file-io/metadata-test
rm -rf metadata-test
```

복사된 디렉터리에서 `file-info`를 실행하자 `original.txt`와 `hard-link.txt`의 inode가 서로 다르게 나타났다.

처음에는 프로그램의 출력 오류를 의심했지만, `ls -li`와 `stat`에서도 서로 다른 inode와 link count 1이 확인됐다.

원인은 일반적인 `cp -r`이 두 이름의 hard link 관계를 보존하지 않고 각각의 파일 내용을 별도로 복사했기 때문이다.

```text
복사 전:

original.txt ───┐
                ├──→ 같은 inode
hard-link.txt ──┘

cp -r 이후:

original.txt  → 새 inode A
hard-link.txt → 새 inode B
```

파일 내용은 같았지만 서로 독립적인 일반 파일이 됐다.

Hard link 관계까지 보존하며 복사하려면 다음과 같은 방법을 사용할 수 있다.

```bash
cp -a SOURCE_DIRECTORY DESTINATION
```

또는 다음 옵션을 사용할 수 있다.

```bash
cp -r --preserve=links SOURCE_DIRECTORY DESTINATION
```

현재 파일을 삭제하지 않고 hard link 동작을 다시 확인하기 위해 새로운 link를 만들었다.

```bash
ln metadata-test/original.txt metadata-test/hard-test.txt
```

그 결과 `original.txt`와 `hard-test.txt`의 inode가 같고 link count가 2인 것을 확인했다.

이번 사례를 통해 파일 내용이 같다는 사실과 같은 inode를 공유한다는 사실은 서로 다른 개념임을 확인했다.

## strace로 시스템 콜 확인하기

다음 명령으로 `lstat()`이 실제로 어떤 시스템 콜을 사용하는지 확인했다.

```bash
strace -e trace=newfstatat \
./file-info metadata-test/symbolic-link.txt
```

관련 출력은 다음과 같았다.

```text
newfstatat(AT_FDCWD, "metadata-test/symbolic-link.txt",
    {st_mode=S_IFLNK|0777, st_size=12, ...},
    AT_SYMLINK_NOFOLLOW) = 0
```

각 값의 의미는 다음과 같다.

```text
AT_FDCWD
→ 현재 작업 디렉터리를 기준으로 상대 경로 해석

AT_SYMLINK_NOFOLLOW
→ 마지막 심볼릭 링크를 따라가지 않음

S_IFLNK | 0777
→ 파일 종류는 symbolic link, 권한 비트는 0777

st_size = 12
→ 링크가 저장한 경로 문자열의 크기

반환값 0
→ 메타데이터 조회 성공
```

소스 코드에서는 `lstat()`을 호출했지만 Linux의 libc는 내부적으로 더 일반적인 `newfstatat()` 시스템 콜을 사용할 수 있다.

Day 2와 Day 3에서 코드의 `open()` 호출이 `strace`에서 `openat()`으로 나타났던 것과 비슷한 wrapper 구조다.

## 구현 중 발생한 문제

### stat(3type) 문서 찾기

`man 2 stat`에서 `stat(3type)` 문서를 참고하라는 내용을 확인했지만 처음에는 여는 방법을 몰랐다.

```bash
man 3type stat
```

같은 이름의 모든 매뉴얼 경로는 다음 명령으로 확인할 수 있다.

```bash
man -aw stat
```

### errno의 의미

처음에는 `lstat()`이 실패하면 `-1`과 `errno`를 반환한다고 표현했다.

정확히는 `lstat()`이 -1을 반환하고 별도의 오류 상태인 `errno`를 설정한다.

### switch의 default 누락

처음 구현한 `switch`에는 일반 파일, 디렉터리, 심볼릭 링크만 있었고 `default`가 없었다.

이 경우 다른 종류의 파일을 입력하면 종류를 출력하지 않고 성공 상태로 종료할 수 있다. 알 수 없는 종류를 `other`로 출력하도록 `default`를 추가했다.

### hard link가 서로 다른 inode로 출력됨

처음에는 `file-info`의 inode 출력 형식이나 캐스팅이 잘못된 것으로 의심했다.

실제 파일 상태와 관련 셸 명령을 확인한 결과, `cp -r` 과정에서 hard link 관계가 보존되지 않은 것이 원인이었다.

코드의 출력이 기존 Linux 도구와 일치하는지 `stat`과 `ls -li`로 교차 확인하는 것이 원인 분석에 도움이 됐다.

## 남은 개선 사항

현재 프로그램은 핵심 메타데이터만 출력한다.

향후 다음 정보를 추가할 수 있다.

- `st_uid`와 `st_gid`
- 마지막 접근 시각과 변경 시각
- block size와 할당된 block 수
- setuid, setgid, sticky bit
- character device, block device, socket, FIFO 세부 구분
- `rwxr-xr-x` 형태의 권한 출력
- symbolic link가 가리키는 경로 출력
- `stat()`과 `lstat()` 선택 옵션

또한 Day 3에서 구현한 `mini-cp`에 `fstat()`을 적용할 수 있다.

원본과 대상을 열고 다음 두 값을 비교하면 서로 다른 이름이 같은 파일이나 hard link를 가리키는지 확인할 수 있다.

```text
st_dev
st_ino
```

이를 이용하면 원본과 대상이 같은 inode인데 `O_TRUNC`로 내용을 지우는 문제를 방지할 수 있다.

이번 실습에서는 `lstat()`을 이용해 파일 종류와 메타데이터를 조회하고, inode를 기준으로 hard link와 symbolic link의 차이를 관찰했다.

특히 `cp -r`이 파일 내용은 복사하지만 hard link 관계는 기본적으로 보존하지 않는다는 것을 실제 사례를 통해 확인했다.
