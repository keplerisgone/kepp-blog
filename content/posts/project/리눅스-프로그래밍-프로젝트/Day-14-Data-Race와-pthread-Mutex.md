---
title: "Day 14: Data Race와 pthread Mutex"
date: "2026-08-29T14:12:54+09:00"
lastmod: "2026-08-29T14:12:54+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-14-data-race와-pthread-mutex/"
weight: 14
ShowToc: true
---

여러 worker thread가 하나의 counter를 동시에 증가시켜 data race와 lost update를 재현하고, `pthread_mutex_t`로 critical section을 보호했다.

```text
Worker 4개 × 1,000,000회 증가
Expected: 4,000,000
```

Mutex가 없는 실행에서는 실제 결과가 `1,532,081`로 달라졌고, mutex 적용 후에는 `4,000,000`이 출력됐다.

## counter++와 lost update

`counter++`는 C source에서는 한 문장이지만 개념적으로 read-modify-write 과정이다.

```text
1. counter 읽기
2. 1 더하기
3. counter에 저장
```

두 thread의 실행이 겹치면 증가 결과가 사라질 수 있다.

```text
counter = 10

Thread A: 10 읽기
Thread B: 10 읽기
Thread A: 11 저장
Thread B: 11 저장

Expected: 12
Actual:   11
```

여러 thread가 같은 memory location에 접근하고 하나 이상이 값을 변경하는데 synchronization이 없다면 data race가 발생한다. C에서 data race는 단순한 오차가 아니라 undefined behavior다.

결과가 한 번 우연히 `4,000,000`으로 나오더라도 코드가 안전하다는 증거가 되지 않는다.

## volatile은 해결책이 아니다

`volatile`은 memory access가 compiler에 의해 함부로 제거되지 않도록 표현하는 데 사용한다. 그러나 다음 성질은 제공하지 않는다.

- Atomicity
- Mutual exclusion
- Thread 간 실행 순서
- Memory synchronization

따라서 shared counter에 `volatile`만 추가해도 data race는 해결되지 않는다.

## Critical section과 Mutex

동시에 실행되면 안 되는 shared-data 접근 영역을 critical section이라고 한다.

이번 코드에서는 `counter++`만 critical section이다.

```text
pthread_mutex_lock()
→ counter++
→ pthread_mutex_unlock()
```

한 thread가 mutex를 보유하는 동안 다른 thread는 같은 mutex를 획득할 수 있을 때까지 기다린다.

Mutex의 lifecycle은 다음 순서로 구성했다.

```text
pthread_mutex_init() 한 번
→ worker thread 생성
→ worker가 lock / counter++ / unlock
→ 모든 worker pthread_join()
→ pthread_mutex_destroy() 한 번
```

처음에는 `pthread_mutex_init()`을 thread 생성 반복문 안에 넣었다. 이미 초기화됐거나 사용 중인 mutex를 다시 초기화하는 것은 undefined behavior이므로 생성 반복문 앞으로 이동했다.

## Default mutex attributes

다음 호출은 기본 mutex attribute를 사용한다.

```c
pthread_mutex_init(&mutex, NULL);
```

이번 실습에서는 같은 프로세스의 thread들이 사용하는 `PTHREAD_MUTEX_DEFAULT` mutex로 이해했다. Process-shared, robust, recursive, priority protocol 같은 특수 설정은 사용하지 않았다.

정적 초기화도 가능하다.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

차이는 다음과 같다.

```text
PTHREAD_MUTEX_INITIALIZER
→ 선언 시 기본 mutex 정적 초기화
→ 별도 init 함수 호출과 반환값 없음

pthread_mutex_init()
→ 실행 중 동적 초기화
→ 반환값 검사 가능
→ custom attribute 전달 가능
```

## pthread mutex의 error handling

Mutex 관련 pthread 함수는 성공 시 0, 실패 시 error number를 직접 반환한다.

```text
성공 → 0
실패 → error number
```

`errno`에 의존해 `perror()`를 호출하지 않고 반환값을 `strerror()`로 변환해야 한다.

```text
err = pthread_mutex_lock(...)
err != 0
→ fprintf(stderr, "%s", strerror(err))
```

Lock에 실패했다면 mutex를 획득하지 못했으므로 해당 경로에서 unlock하면 안 된다.

현재 실습 코드에는 `pthread_mutex_init()`, `lock()`, `unlock()`, `destroy()`의 반환값 검사를 추가하는 작업이 TODO로 남아 있다.

## Mutex destroy 시점

`pthread_mutex_destroy()`는 mutex를 사용할 thread가 더 이상 없을 때 호출해야 한다.

단순히 특정 순간 mutex가 unlocked인 것만으로는 충분하지 않다. 아직 실행 중인 worker가 이후 lock을 시도할 수 있기 때문이다.

```text
모든 worker join
→ mutex를 사용할 thread가 없음을 보장
→ mutex destroy
```

Locked 상태이거나 다른 thread가 사용하거나 기다리는 mutex를 destroy하는 것은 undefined behavior다.

## 실행 결과 비교

Mutex 없는 버전:

```text
expected: 4000000
actual:   1532081
```

Mutex 적용 버전:

```text
expected: 4000000
actual:   4000000
```

Mutex 버전은 정확해졌지만 느려졌다. Counter 증가 4백만 회 각각에 lock과 unlock을 수행하고 여러 worker가 하나의 mutex를 두고 경쟁하기 때문이다.

```text
정확성 획득
↔ synchronization과 contention 비용 발생
```

## ThreadSanitizer

Data race를 동적으로 검출하기 위해 ThreadSanitizer를 사용했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -O1 \
    -fsanitize=thread -fno-omit-frame-pointer -pthread \
    -o race-demo-tsan race-demo.c
```

현재 환경에서는 일반 실행 시 다음 오류가 발생했다.

```text
FATAL: ThreadSanitizer: unexpected memory mapping
```

TSan이 사용하는 shadow memory 영역과 실행 시 주소 배치가 충돌한 환경 문제였다. 해당 자식 프로세스에만 ASLR을 비활성화해 실행했다.

```bash
setarch x86_64 -R ./race-demo-tsan
setarch x86_64 -R ./mutex-demo-tsan
```

System-wide ASLR 설정은 변경하지 않았다.

Race 버전에서는 다음 정보가 보고됐다.

```text
WARNING: ThreadSanitizer: data race
Write by thread T1
Previous read/write by another thread
Location is global 'counter'
race-demo.c의 counter++ line
```

Mutex 버전에서는 같은 counter에 대한 data race warning이 나타나지 않았고 결과도 `4,000,000`이었다.

TSan은 실행 timing과 scheduling에 영향을 주므로 sanitizer 실행에서 나온 actual counter 숫자 자체보다 충돌한 memory location과 source line을 확인하는 것이 중요하다.

## futex와 fast path

다음 명령으로 thread 생성, join과 mutex 경쟁을 관찰했다.

```bash
strace -f -c \
    -e trace=clone,clone3,futex \
    ./mutex-demo
```

Lock과 unlock은 각각 4백만 번 호출했지만 `futex` system call은 그보다 훨씬 적었다.

```text
경쟁 없음
→ user-space atomic operation으로 처리하는 fast path

실제 대기 필요
→ futex를 이용해 kernel에서 재우고 깨우는 slow path
```

관찰된 `futex`가 모두 counter mutex에서 발생한 것은 아니다. `pthread_join()`도 worker 종료를 기다리기 위해 futex를 사용할 수 있다.

## Linux System Software 직무와의 연결

`sysmon-agent`에서는 collector thread가 shared metrics를 갱신하는 동안 network thread가 같은 metrics를 읽게 된다.

```text
Collector Thread ── write ──┐
                            ├─ Shared Metrics + Mutex
Network Thread ─── read ────┘
```

Mutex가 없다면 network thread가 일부 field만 새 값인 불완전한 snapshot을 전송할 수 있다. Critical section의 범위와 mutex lifecycle을 올바르게 설계하는 능력이 필요하다.

## 학습 정리

### Learned

- `counter++`는 atomic operation이 아니라 read-modify-write다.
- Synchronization 없는 shared counter 접근은 data race이며 undefined behavior다.
- 실행 결과가 우연히 맞아도 thread-safe라는 뜻은 아니다.
- `volatile`은 atomicity나 thread synchronization을 제공하지 않는다.
- Mutex로 critical section을 보호하면 lost update를 방지할 수 있다.
- Mutex는 한 번 초기화하고 모든 worker를 join한 뒤 destroy해야 한다.
- Mutex의 uncontended path는 user-space에서 처리될 수 있다.
- 실제 대기와 join에는 Linux의 `futex`가 사용될 수 있다.
- ThreadSanitizer로 충돌한 thread, global variable과 source line을 찾을 수 있다.

### APIs

- `pthread_mutex_init()`
- `pthread_mutex_lock()`
- `pthread_mutex_unlock()`
- `pthread_mutex_destroy()`
- `pthread_create()`
- `pthread_join()`

### Commands

- `man 3posix pthread_mutex_init`
- `man 3posix pthread_mutex_lock`
- `gcc -pthread`
- `gcc -fsanitize=thread`
- `setarch x86_64 -R`
- `strace -f -c -e trace=clone,clone3,futex`

### Mistakes

- `pthread_mutex_init()`을 worker 생성 반복문 안에 넣어 같은 mutex를 여러 번 초기화했다.
- 처음에는 pthread 함수 반환값을 `(void)err`로 버려 compiler warning만 숨겼다.
- 정상 출력만으로 concurrency code의 안전성을 판단할 수 있다고 생각할 수 있었다.
- TSan의 `unexpected memory mapping`을 처음에는 코드의 data race 진단으로 오해할 수 있었다.

### TODO

- 불필요한 `<threads.h>` header를 제거한다.
- Mutex init, lock, unlock, destroy의 반환값을 모두 검사한다.
- Thread 생성이 일부만 성공했을 때 실패 상태를 반환한다.
- Join 하나가 실패해도 나머지 생성된 thread의 회수를 계속 시도한다.
- 고정 숫자를 thread count 상수로 통일한다.

### Next

- Socket의 endpoint와 file descriptor 개념을 학습한다.
- Blocking TCP echo server와 client를 구현한다.
- 이후 `/proc` collector와 `sysmon-agent`로 연결한다.
