---
title: "Day 1 - 명령행 인자와 파일 입출력"
date: "2026-07-25T13:08:00+09:00"
lastmod: "2026-07-25T13:08:00+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 프로젝트의 목표는 더 멋진 리눅스 소프트웨어 프로그래머가 되기 위해서 C와 리눅스의 명령에 익숙해지는 것이다.
첫 번째 날은 <이거 왜 필요한건지?>

실험 환경은 rocky 

---

## 개발 환경 확인

우선, 프로젝트를 진행하기 위해 다음과 같은 개발 도구들이 필요하므로, 명령어를 통해 개발 도구들의 버전을 확인하자. 

```bash
gcc --version
gdb --version
valgrind --version
strace --version
git --version
```

- gcc는 c 언어의 컴파일러이다.
- gdb는 c 언어의 디버깅 도구이다.
- valgrind는 메모리 사용을 탐지하는 도구이다.
- strace는 <모르겠다>
- git은 모두가 알고 있는 버전 관리 도구이다.

버전 확인 결과는 다음과 같다.

<결과>

현재 valgrind 말고는 모두 잘 설치되어있으므로, 다음 명령을 통해 valgrind를 설치하도록 한다. rocky에서는 패키지 관리자로 dnf를 사용한다.

```bash
sudo dnf install valgrind
```

만약 다른 OS를 사용하고 있을 경우 사용하는 패키지에 맞게 명령어를 실행하도록 하자.

## 작업 디렉토리 확인

디렉토리 구조는 다음처럼 짜기로 했다.

```
linux-system-practice/
  ├── .gitignore
  └── file-io/
      ├── args.c
      ├── read-file.c
      └── test.txt

.gitignore:

*.o
*.out
build/
node_modules/
file-io/args
file-io/read-file
```

컴파일 결과로 생성되는 args와 read-file 실행 파일은 Git에서 제외했다.

## argc와 argv 연습

argc와 argv는 main 함수를 작성하면서 많이 보게될 인자인데, 여기에는 프로그램을 실행할 때 전달돠는 명령행 인자들과 관련된 정보가 저장되는 곳이다.

```c
// args.c
#include <stdio.h>

int main(int argc, char *argv[]){
printf("argc = %d\n", argc);
for (int i=0; i < argc; i++) {
	printf("argv[%d] = %s\n", i, argv[i]);
}

return 0;
}
```

위는 명령행 인자를 받아 모두 출력하는 코드이다. 컴파일에 사용되는 옵션에 대해서는 아래에서 더 자세히 다룬다.

```bash
// 컴파일
gcc -Wall -Wextra -g args.c -o args
// 아무런 메세지도 출력되지 않음 (에러가 0)

// 실행
./args hello hi von vonyage

// 실행 결과
argc = 5
argv[0] = ./args
argv[1] = hello
argv[2] = hi
argv[3] = von
argv[4] = voyage
```

위를 통해 알게 된 사실은 다음과 같다.
- `argc`는 명령행 인자의 개수다.
- 프로그램을 호출한 이름 또는 경로도 포함되므로 값은 5다.
- `argv`는 명령행 인자 문자열들을 담은 배열이다.
- `argv[0]`에는 일반적으로 프로그램을 호출한 이름이나 경로가 들어간다.
- 사용자가 전달한 첫 번째 인자는 `argv[1]`에 들어간다.

나처럼 이상하게 인자에 argc[i]를 넣는 실수를 하지 말자.

## 표준 파일 입출력 연습

다음은 명령행 인자로 파일 이름을 받아 파일 내용을 터미널에 출력하는 프로그램을 작성했다.
테스트로 사용되는 입력 텍스트 파일은 다음과 같다.

```txt
// 테스트 파일
test.txt:

hello
hi
von
voyage
```

다음은 소스 코드이다.

```c
#include <stdio.h>

int main(int argc, char* argv[]){
if ( argc != 2 ) {
	printf("usage: ./read-file [file name]\n");
	return 0;
}

FILE* pFile = fopen(argv[1], "r");
if ( !pFile ) {
	printf("no file\n");
	return 0;
}

char str[100];
while( fgets(str, 100, pFile )) {
	printf("%s", str);
}

fclose(pFile);

return 0;
}
```

중요한 함수들은 `fopen(), fgets()`이다.

### 컴파일

gcc -Wall -Wextra -g read-file.c -o read-file

컴파일 결과:

warning 0
error 0

### 정상 실행

./read-file test.txt

실행 결과:

hello
hi
von
voyage

### 인자 없이 실행

./read-file

실행 결과:

usage: ./read-file [file name]

### 존재하지 않는 파일 실행

./read-file nonexistent-file.txt

실행 결과:

no file

## 파일 입출력에서 배운 내용

### fopen()

성공: FILE * 반환
실패: NULL 반환

현재 프로그램은 파일을 읽기 전용 모드로 열었다.

fopen(argv[1], "r")

### fgets()

fgets()는 다음 조건 중 하나를 만족할 때까지 데이터를 읽는다.

- 줄바꿈 문자를 읽었을 때
- 지정한 버퍼 크기보다 하나 적은 수의 문자를 읽었을 때
- EOF에 도달했을 때

성공하면 전달받은 버퍼의 주소를 반환하고, EOF 또는 읽기 오류가 발생하면 NULL을 반환한다.
현재 버퍼 크기는 100이므로 한 번에 최대 99개의 문자와 문자열 종료 문자 '\0'을 저장할 수 있다.

### fclose()

fclose()는 파일 스트림이 사용하던 자원을 정리한다.

- 운영체제의 파일 디스크립터 반환
- 표준 입출력 버퍼 정리
- 파일 스트림의 내부 상태 정리
- 출력 스트림이라면 남은 버퍼 내용 반영

malloc()으로 할당한 메모리를 free()하는 것과 마찬가지로, 사용한 자원을 해제한다는 공통점이 있다.

## 작성하면서 수정한 문제

처음 작성한 코드에는 다음 문제가 있었다.

### 읽기 모드 문자열

fopen()의 모드를 r로 작성했지만, 두 번째 인자는 문자열이어야 하므로 "r"로 수정했다.

### 잘못된 반복 조건

처음에는 다음과 같은 형태로 버퍼 자체를 검사했다.

`while (str)`

문자 배열의 주소는 항상 존재하므로 이 조건은 계속 참이 된다. 따라서 fgets()의 반환값을 반복 조건으로 검사하도록 변경했다.

### 변수 이름의 대소문자

C는 대소문자를 구분하므로 pFile과 pfile은 서로 다른 변수다. 선언할 때 사용한 이름과 동일하게
맞췄다.

### 파일 닫기 누락

파일 읽기가 끝난 뒤 fclose()를 호출하도록 수정했다.

## GCC 컴파일 옵션

사용한 컴파일 명령:

`gcc -Wall -Wextra -g 소스파일.c -o 실행파일`

- `-Wall`: 주요 컴파일 경고를 활성화한다. 이름과 달리 모든 경고를 활성화하는 것은 아니다.

- `-Wextra`: -Wall에 포함되지 않은 추가 경고를 활성화한다.

- `-g`: 실행 파일에 소스 줄 번호, 변수 이름, 함수 정보 등의 디버깅 정보를 포함한다. GDB에서 활용할 수 있다.

- `-o`: 생성할 실행 파일의 이름을 지정한다.

## Day 1 정리

완료한 항목:

[x] 개발 도구 상태 확인
[x] 학습 저장소 구조 확인
[x] args.c 직접 작성
[x] args.c 컴파일 및 실행
[x] argc와 argv 이해
[x] read-file.c 직접 작성
[x] read-file.c 컴파일 및 실행
[x] 파일이 없을 때의 예외 처리
[x] 파일 스트림 닫기
[x] -Wall -Wextra -g 이해
[x] 경고와 오류 없이 컴파일
[x] Git 커밋 및 GitHub push
