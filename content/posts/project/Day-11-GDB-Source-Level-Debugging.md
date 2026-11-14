---
title: "Day 11: GDB Source-Level Debugging"
date: "2026-08-17T17:20:08+09:00"
lastmod: "2026-08-17T17:20:08+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 GDB로 프로그램의 실행을 중단하고, 소스 줄 단위로 진행하면서 함수 인자, 지역 변수, raw memory와 call stack을 관찰하는 것이다.

정상적인 배열 합계·평균 프로그램을 먼저 추적한 뒤 의도적으로 off-by-one을 추가했다. 범위 밖 접근이 반드시 crash하지 않는다는 점을 확인하고, GDB에서 index를 크게 변경해 `SIGSEGV`를 재현한 뒤 `backtrace`로 원인을 조사했다.

```text
정상 흐름 관찰
→ off-by-one 추가
→ 잘못된 index 확인
→ SIGSEGV 재현
→ backtrace와 memory 검사
→ 원인 수정
→ 재빌드 및 검증
```

## Debug symbol과 최적화

다음 옵션으로 실습 프로그램을 컴파일했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -O0 \
    -o gdb-lab gdb-lab.c
```

각 옵션의 역할은 다음과 같다.

```text
-g  → 실행 주소를 소스 파일, 함수, 줄 번호, 변수 정보와 연결하는 debug symbol 추가
-O0 → compiler 최적화를 비활성화해 소스 흐름과 실행을 대응시키기 쉽게 함
```

`-g`는 오류 검사를 활성화하는 옵션이 아니다. Compiler가 만든 기계어와 source-level 정보를 GDB가 연결할 수 있도록 metadata를 포함한다.

최적화된 프로그램에서는 변수가 제거되거나 여러 소스 줄이 합쳐지고 실행 순서가 달라질 수 있다. 실제 release binary도 디버깅해야 하지만, GDB 입문에서는 `-O0`으로 기본 동작을 관찰했다.

## gdb-lab 프로그램

프로그램의 함수 구조는 다음과 같다.

```text
main()
└─ print_report()
   └─ calculate_avg()
      └─ calculate_sum()
```

최종 코드는 다음과 같다.

```c
#include <stdio.h>

int calculate_sum(const int num[], int cnt){
	int sum = 0;
	for (int i=0; i< cnt; i++) {
		sum += num[i];
	}
	return sum;
}

float calculate_avg(const int num[], int cnt){
	return calculate_sum(num, cnt) / (float) cnt;
}

void print_report(const int num[], int cnt){
	float avg = calculate_avg(num, cnt);
	printf("average: %f\n", avg);
}

int main(void) {
	const int num[5] = {10, 20, 30, 40, 50};
	print_report(num, 5);
	return 0;
}
```

최종 실행 결과는 다음과 같았다.

```text
average: 30.000000
```

Compiler warning은 없었고 종료 상태는 0이었다.

## GDB 시작과 breakpoint

다음 명령으로 GDB를 시작했다.

```bash
gdb ./gdb-lab
```

`main()`에 breakpoint를 설정하고 실행했다.

```text
break main
run
```

Breakpoint는 프로그램을 종료하는 것이 아니라 해당 위치에 도달했을 때 실행을 일시 정지한다. 정지 상태에서 변수, memory, stack을 관찰한 뒤 다시 실행할 수 있다.

주요 명령은 다음과 같다.

```text
break       → breakpoint 설정
run         → 프로그램 시작
next        → 함수 내부로 들어가지 않고 다음 소스 줄
step        → 호출하는 함수 내부로 진입
continue    → 다음 breakpoint 또는 종료까지 계속 실행
finish      → 현재 함수가 반환할 때까지 실행
```

## 최신 source와 실행 파일의 불일치

`main(int argc, char *argv[])`를 `main(void)`로 수정한 뒤 실행 파일을 다시 빌드하지 않고 GDB를 실행해 다음 경고가 나타났다.

```text
warning: Source file is more recent than executable.
```

GDB는 이전 실행 파일의 debug symbol에서 `argc`와 `argv`를 표시했지만 화면의 source는 최신 `main(void)`를 보여줬다.

```text
Source 수정
→ 실행 파일과 debug symbol은 이전 상태
→ 줄 번호와 변수 정보가 최신 source와 불일치
```

GDB를 종료하고 다시 컴파일한 뒤 새 실행 파일로 재시작해 해결했다.

```text
source 수정
→ rebuild
→ GDB 재시작
```

Make/CMake 같은 build system이 변경된 source를 추적해 다시 빌드하는 이유와도 연결된다.

## 현재 실행 위치 확인

GDB가 정지한 위치는 다음 명령으로 확인했다.

```text
frame
list .
backtrace
```

```text
frame     → 현재 선택된 stack frame과 다음 source line
list .    → 현재 실행 위치 주변 source
backtrace → 현재 함수부터 최초 호출자까지의 call stack
```

GDB가 표시하는 source line은 일반적으로 이미 실행한 줄이 아니라 다음에 실행할 줄이다.

## 초기화 전 지역 변수

`main()` 진입 직후 `info locals`를 실행하자 배열이 다음처럼 보였다.

```text
num = {0, 0, -134325520, 32767, -9376}
```

현재 위치는 배열 초기화 줄을 실행하기 전이었다.

```c
const int num[5] = {10, 20, 30, 40, 50};
```

지역 배열의 stack 공간은 존재했지만 초기화 대입이 아직 실행되지 않아 이전 stack memory의 의미 없는 값이 보였다.

첫 `next`는 초기화 줄을 실행하기 직전에 멈췄고, 다시 `next`를 실행해 해당 줄을 지난 뒤 배열이 정상적으로 나타났다.

```text
num = {10, 20, 30, 40, 50}
```

`calculate_sum()`에 진입한 직후 `sum`도 초기화 전에는 큰 의미 없는 값을 표시했다. `int sum = 0;` 줄을 `next`로 실행한 뒤 0으로 바뀌었다.

이를 통해 breakpoint와 source line에서 멈춘 시점이 해당 줄 실행 전이라는 점을 확인했다.

## next와 step

`main()`에서 `next`를 사용해 배열 초기화 전후를 비교했다.

`print_report()` 호출 줄에서는 `step`을 사용해 함수 내부로 들어갔다.

```text
next
→ 함수 호출 전체를 한 줄처럼 실행

step
→ 호출 함수의 첫 source line으로 진입
```

각 함수에서 다음 명령으로 상태를 관찰했다.

```text
info args
info locals
print expression
display expression
```

```text
info args   → 호출자가 현재 함수에 전달한 인자
info locals → 현재 frame의 지역 변수
print       → 현재 한 번 표현식 평가
display     → 프로그램이 멈출 때마다 표현식 자동 출력
```

정상 반복에서는 합계가 다음처럼 변했다.

```text
i=0, sum=0
i=1, sum=10
i=2, sum=30
i=3, sum=60
i=4, sum=100
최종 sum=150
```

## Conditional breakpoint

정상 동작을 확인한 뒤 반복 조건을 의도적으로 `i <= cnt`로 변경해 off-by-one 오류를 만들었다.

```text
유효 index → 0, 1, 2, 3, 4
잘못된 index → 5
```

반복문 본문의 source line에 조건부 breakpoint를 설정했다.

```text
break gdb-lab.c:LINE if i == cnt
continue
```

GDB는 정상 index 0부터 4까지는 멈추지 않고 `i == 5`일 때만 중단했다.

```text
print i
print cnt
print sum
print num[i]
```

결과:

```text
i      = 5
cnt    = 5
sum    = 150
num[5] = 32767
```

`sum`은 범위 밖 값을 더하기 직전이므로 정상 원소의 합계 150이었다. `num[5]`는 배열 바깥의 stack memory에서 우연히 읽힌 32767이었다.

## Undefined behavior는 항상 crash하지 않는다

Off-by-one 상태에서 `finish`를 실행해도 segmentation fault가 발생하지 않았다.

C 관점에서 `num[5]`는 배열 범위 밖이므로 undefined behavior다. 그러나 CPU와 kernel은 C 배열의 원소 개수를 알지 못한다.

```text
CPU가 계산한 주소가 mapped되고 읽기 가능
→ 어떤 값을 읽음
→ crash하지 않을 수 있음

주소가 mapping되지 않았거나 권한이 없음
→ SIGSEGV
```

따라서 범위 밖 접근은 다음 중 어느 결과든 만들 수 있다.

```text
segmentation fault
잘못된 계산 결과
우연히 정상처럼 보이는 결과
다른 memory 손상
```

GDB가 `num[5]`의 값을 표시했다는 것은 그 주소에서 memory를 읽을 수 있었다는 뜻일 뿐 유효한 배열 원소라는 의미가 아니다.

범위 밖 접근 자체를 자동 탐지하는 목적에는 이후 학습할 AddressSanitizer와 Valgrind가 더 적합하다.

## GDB에서 SIGSEGV 재현

Memory fault와 backtrace를 관찰하기 위해 조건부 breakpoint에서 index 값을 크게 변경했다.

```text
set variable i = 100000000
next
```

`num + i`로 계산한 주소가 process에 mapping되지 않은 영역을 가리켜 GDB가 `SIGSEGV`에서 실행을 중단했다.

```text
Program received signal SIGSEGV, Segmentation fault.
calculate_sum (...) at gdb-lab.c:6
```

이는 source의 원래 입력에서 자연스럽게 발생한 crash가 아니라 GDB 조작 학습을 위해 상태를 인위적으로 변경한 실험이다.

## Raw memory 확인

배열이 저장된 memory를 다음 명령으로 확인했다.

```text
x/5dw num
```

명령의 의미:

```text
x    → examine memory
/5   → 5개
d    → signed decimal
w    → GDB word, 4 bytes
num  → 읽기 시작 주소
```

결과:

```text
0x7fffffffda60: 10  20  30  40
0x7fffffffda70: 50
```

현재 환경에서 `int`는 4 bytes이므로 주소는 다음처럼 증가한다.

```text
0x7fffffffda60 → num[0]
0x7fffffffda64 → num[1]
0x7fffffffda68 → num[2]
0x7fffffffda6c → num[3]
0x7fffffffda70 → num[4]
```

함수 매개변수의 `num[]`은 pointer로 전달되므로 `main()`의 배열 전체를 함수마다 복사하지 않는다. 모든 frame에서 같은 시작 주소 `0x7fffffffda60`을 가리켰다.

## Backtrace

`SIGSEGV` 상태에서 다음 call stack을 확인했다.

```text
#0 calculate_sum (num=0x7fffffffda60, cnt=5) at gdb-lab.c:6
#1 calculate_avg (num=0x7fffffffda60, cnt=5) at gdb-lab.c:12
#2 print_report (num=0x7fffffffda60, cnt=5) at gdb-lab.c:16
#3 main () at gdb-lab.c:22
```

Backtrace는 현재 함수부터 호출자를 거슬러 올라간다.

```text
#3 main
 ↓
#2 print_report
 ↓
#1 calculate_avg
 ↓
#0 calculate_sum
    SIGSEGV 발생
```

Frame 0에서 `info locals`를 확인하면 비정상 index와 합계를 볼 수 있다. `frame N`, `up`, `down`으로 선택 frame을 변경해 각 호출자의 인자와 지역 변수를 관찰할 수 있다.

```text
frame 0 → SIGSEGV가 발생한 calculate_sum
frame 1 → calculate_sum의 반환을 기다리는 calculate_avg
frame 2 → 평균 계산을 기다리는 print_report
frame 3 → 최초 호출자 main
```

Backtrace 앞의 `0x5555...` 계열 주소는 실행 코드 영역의 instruction address이고 `num=0x7fff...`는 stack에 있는 배열 data address였다.

## GDB와 editor 사용

현재 실행 위치를 확인하는 기본 명령은 다음과 같다.

```text
frame
list .
backtrace
```

Terminal 안에서 source를 계속 보려면 GDB TUI를 사용할 수 있다.

```bash
gdb -tui ./gdb-lab
```

```text
layout src
layout split
tui disable
```

또한 tmux에서 Neovim과 GDB를 나란히 실행하는 방식도 사용할 수 있다.

```text
tmux
├─ nvim gdb-lab.c
└─ gdb ./gdb-lab
```

Linux System Software와 Embedded Linux에서는 SSH, remote target, core dump와 같이 GUI가 없는 환경이 많아 GDB CLI 기본기를 알아야 한다. Editor integration이나 IDE를 사용해도 내부 debugger backend가 GDB인 경우가 많다.

## 최종 복구와 검증

실습 후 반복 조건을 정상적인 `i < cnt`로 복구했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -O0 \
    -o gdb-lab gdb-lab.c

./gdb-lab
echo $?
```

결과는 평균 30.0, compiler warning 0, 종료 상태 0이었다.

## 남은 개선 사항

- `cnt == 0`일 때 평균 계산에서 0으로 나누지 않도록 처리한다.
- 배열 크기에 더 적합한 `size_t` 사용을 검토한다.
- `main()`에서 배열 길이 5를 반복해 직접 쓰지 않고 배열 자체에서 계산한다.
- 함수와 반복문의 공백 style을 일관되게 정리한다.
- 필요하면 함수들을 현재 translation unit 내부로 제한하는 `static`을 사용한다.
- AddressSanitizer와 Valgrind로 out-of-bounds와 memory 오류를 자동 탐지한다.

## Linux System Software 직무와의 연결

GDB는 crash 지점만 확인하는 도구가 아니다. 장시간 실행되는 daemon, process worker와 thread가 현재 어느 함수에서 어떤 상태로 멈췄는지 조사하고 core dump를 사후 분석하는 데 사용한다.

Source line, 변수, raw memory, call stack을 연결해 원인을 추론하는 능력은 `sysmon-agent`의 crash와 잘못된 metric 계산을 분석할 때 직접 사용된다.

## 학습 정리

### Learned

- `-g`는 instruction address와 source 정보를 연결하는 debug symbol을 포함한다.
- Breakpoint는 source line 실행 전에 프로그램을 일시 정지한다.
- `next`와 `step`은 함수 내부 진입 여부가 다르다.
- 초기화 전 지역 변수에는 의미 없는 stack 값이 보일 수 있다.
- Conditional breakpoint로 문제 상태에만 즉시 멈출 수 있다.
- Undefined behavior는 반드시 segmentation fault를 발생시키지 않는다.
- `backtrace`와 frame 이동으로 crash까지의 호출 경로를 조사할 수 있다.
- `x` 명령으로 pointer가 가리키는 raw memory를 확인할 수 있다.

### Commands

- `break` / `info breakpoints` / `delete`
- `run` / `next` / `step` / `continue` / `finish`
- `print` / `display`
- `info args` / `info locals`
- `backtrace` / `frame` / `up` / `down`
- `list .`
- `set variable`
- `x/5dw`
- `layout src`

### Mistakes

- Source를 수정한 뒤 실행 파일을 다시 빌드하지 않아 source와 debug symbol이 불일치했다.
- 다음에 실행할 source line을 이미 실행된 줄로 생각했다.
- 초기화 전 지역 변수의 쓰레기 값을 계산 결과로 오해할 수 있었다.
- 범위 밖 접근이면 항상 segmentation fault가 발생할 것으로 예상했다.

### Next

- Shared memory를 이용한 IPC를 학습한다.
- 이후 pthread debugging에서 `info threads`와 thread별 backtrace를 사용한다.
- AddressSanitizer와 Valgrind를 이용한 memory debugging으로 확장한다.
