---
title: "Day 13: pthread 생성과 join"
date: "2026-08-25T14:22:01+09:00"
lastmod: "2026-08-25T14:22:01+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

이번 실습의 목표는 하나의 프로세스 안에서 worker thread를 생성하고, 구조체를 통해 인자를 전달한 뒤 `pthread_join()`으로 종료를 기다리는 것이었다.

두 worker가 서로 다른 입력을 계산해 각자의 결과 영역에 기록하도록 만들고, Linux에서 thread가 PID와 TID로 어떻게 나타나는지 `ps`, `/proc`, `strace`로 관찰했다.

```text
Main thread
├─ pthread_create() → Worker 1
├─ pthread_create() → Worker 2
├─ pthread_join()   → Worker 1 종료 대기
└─ pthread_join()   → Worker 2 종료 대기
```

## Process와 Thread

서로 다른 프로세스는 독립된 virtual address space를 사용한다. Day 12에서는 두 프로세스가 데이터를 공유하도록 POSIX shared memory object를 별도로 생성하고 mapping했다.

같은 프로세스에 속한 thread는 처음부터 다음 자원을 공유한다.

- Code와 전역 변수
- Heap과 virtual address space
- 열린 file descriptor
- 프로세스의 여러 자원

각 thread는 다음 실행 상태를 별도로 가진다.

- Stack
- Register 상태
- 실행 흐름
- Thread ID

```text
하나의 process address space
├─ Main thread stack
├─ Worker 1 stack
├─ Worker 2 stack
└─ Shared data, heap, code, file descriptors
```

주소 공간이 공유된다는 것은 데이터 전달이 간단하다는 장점이 있지만, 여러 thread가 같은 데이터를 동시에 변경하면 data race가 발생할 수 있다는 뜻이기도 하다.

## pthread_create()

함수 원형은 다음과 같다.

```c
int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```

각 인자의 의미는 다음과 같다.

```text
thread        → 생성된 thread 식별자를 저장할 위치
attr          → thread attribute, 기본값은 NULL
start_routine → 새 thread가 실행할 함수
arg           → start_routine에 전달할 한 개의 void pointer
```

반환값은 일반적인 system call과 다르다.

```text
성공 → 0
실패 → error number를 직접 반환
```

pthread 함수는 일반적으로 실패 시 `-1`과 `errno`를 사용하는 방식이 아니다. 반환된 error number를 보관하고 `strerror()`에 전달했다.

```text
err = pthread_create(...)
err != 0
→ fprintf(stderr, "pthread_create: %s\n", strerror(err))
```

## Start routine

새 thread가 실행할 함수는 다음 형태여야 한다.

```c
void *worker(void *arg);
```

`start_routine`은 함수 포인터이며, 함수 포인터의 포인터가 아니다. `pthread_create()`의 네 번째 인자가 `worker()`의 `arg`로 그대로 전달된다.

이번 worker는 반환값을 별도로 전달하지 않으므로 마지막에 `NULL`을 반환했다.

```text
main의 &info_array[i]
→ pthread_create()의 arg
→ worker(void *arg)
→ struct info *로 해석
```

C에서는 `void *`를 다른 object pointer type으로 암시적으로 변환할 수 있으므로 다음처럼 사용했다.

```c
struct info *info_p = arg;
```

처음에는 `(info *)arg`로 작성했지만 `info`라는 typedef를 선언하지 않았기 때문에 compile error가 발생했다. 선언된 타입 이름은 `struct info`였다.

## Thread 인자 구조체와 lifetime

각 worker에 다음 정보를 전달했다.

```c
struct info {
    int number;
    int input;
    int result;
};
```

Main은 worker 수만큼 구조체 배열을 만들었다.

```text
info_array[0] → Worker 1 전용
info_array[1] → Worker 2 전용
```

반복문 안의 하나의 임시 구조체 주소를 모든 worker에 넘기지 않고, 각 worker가 서로 다른 배열 원소를 가리키도록 했다.

Worker가 포인터를 사용하는 동안 구조체가 유효해야 한다. 이번 구조체 배열은 `main()` stack에 있지만, `main()`이 모든 worker를 join하기 전에는 함수에서 반환하지 않으므로 정상 실행 경로에서 lifetime이 유지된다.

## pthread_join()

`pthread_join()`은 target thread를 종료시키는 함수가 아니다. 지정한 thread가 종료될 때까지 호출한 thread가 기다리고, 종료된 joinable thread를 회수한다.

```c
int pthread_join(pthread_t thread, void **retval);
```

두 번째 인자에는 worker 함수가 반환한 pointer 또는 `pthread_exit()`에 전달한 pointer를 받을 수 있다. 이번에는 반환값을 사용하지 않으므로 `NULL`을 전달했다.

```text
pthread_join(target, NULL)
```

프로세스의 `waitpid()`가 받는 정수 exit status와 달리 `pthread_join()`의 `retval`은 `void *`라는 차이가 있다.

Thread 두 개를 먼저 모두 생성한 뒤, 별도의 반복문에서 모두 join했다.

```text
create Worker 1
create Worker 2
join Worker 1
join Worker 2
결과 출력
```

Create 직후 같은 반복에서 바로 join하면 한 worker가 끝난 뒤 다음 worker를 생성하게 되어 동시 실행의 의미가 줄어든다.

## 실행 순서는 보장되지 않는다

운영체제 scheduler가 어떤 thread를 먼저 실행할지는 보장하지 않는다.

```text
Worker 1 started → Worker 2 started
```

뿐 아니라 다음 순서도 가능하다.

```text
Worker 2 started → Worker 1 started
```

`pthread_join(p_array[0], ...)`을 먼저 호출해도 Worker 1이 Worker 2보다 먼저 끝난다는 뜻은 아니다. Main이 Worker 1을 먼저 기다리기로 선택한 것이다.

각 worker 내부의 실행 흐름은 유지되며, 모든 join 이후에는 worker가 결과를 기록하고 종료한 상태다. 따라서 main은 join 이후 결과를 읽었다.

## 현재 코드가 data race를 만들지 않는 이유

두 worker는 같은 `info_array` 안에 있지만 서로 다른 원소만 수정한다.

```text
Worker 1 → info_array[0].result
Worker 2 → info_array[1].result
```

동일한 memory location에 동시에 접근하지 않으므로 현재 계산에는 data race가 없다. Main은 join이 끝난 후 결과를 읽는다.

다음 실습에서는 여러 thread가 하나의 counter를 동시에 수정하도록 만들어 실제 data race와 lost update를 관찰하고 mutex로 보호한다.

## 구현 코드

```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <string.h>

struct info {
    int number;
    int input;
    int result;
};

void *worker(void *arg) {
    struct info *info_p = arg;
    printf("worker: %d, input: %d\n", info_p->number, info_p->input);

    info_p->result = info_p->input * 2;
    sleep(3);

    printf("worker %d done\n", info_p->number);
    return NULL;
}

int main(void) {
    pthread_t p_array[2];
    struct info info_array[2];
    info_array[0] = (struct info) {1, 10, 0};
    info_array[1] = (struct info) {2, 20, 0};

    int create_cnt = 0;
    for (int i=0; i<2 ; i++) {
        int err = pthread_create(&p_array[i], NULL, worker, &info_array[i]);
        if (err != 0) {
            fprintf(stderr, "pthread_create: %s\n", strerror(err));
            break;
        }
        create_cnt++;
    }

    for (int i=0; i<create_cnt; i++) {
        int err = pthread_join(p_array[i], NULL);
        if (err != 0) {
            fprintf(stderr, "pthread_join: %s\n", strerror(err));
            // TODO: Join the remaining created threads before returning.
            return 1;
        }
    }

    // TODO: Return failure without printing results if not all threads were created.
    printf("worker 1 result: %d\nworker 2 result: %d\n",
           info_array[0].result, info_array[1].result);

    return 0;
}
```

## Compile과 실행

pthread 프로그램은 `-pthread` option을 추가해 compile했다.

```bash
gcc -Wall -Wextra -Wpedantic -g -pthread \
    -o thread-demo thread-demo.c

./thread-demo
```

Compiler warning 없이 build되었고 결과는 다음과 같았다.

```text
worker 1 result: 20
worker 2 result: 40
```

여러 번 실행해도 계산 결과는 같았지만 worker의 출력 순서는 달라질 수 있었다.

## ps와 /proc으로 Thread 관찰

관찰 시간을 확보하기 위해 worker가 잠시 `sleep()`하도록 만들고 background에서 실행했다.

```bash
./thread-demo &
demo_pid=$!

ps -T -p "$demo_pid"
ls /proc/"$demo_pid"/task
grep '^Threads:' /proc/"$demo_pid"/status
wait "$demo_pid"
```

`ps -T`에서 같은 process PID 아래 서로 다른 SPID가 나타났다. `/proc/<PID>/task`에서도 main thread와 worker 두 개에 해당하는 항목 세 개를 확인했다.

```text
Process
├─ Main thread: TID == process PID
├─ Worker 1: 별도의 TID
└─ Worker 2: 별도의 TID
```

Linux kernel은 process와 thread를 모두 task로 관리한다. 같은 프로세스의 thread들은 같은 thread group에 속하지만 각자 다른 TID를 가진다.

## strace로 pthread 내부 관찰

다음 명령으로 thread 생성과 join의 Linux 수준 동작을 관찰했다.

```bash
strace -f \
    -e trace=clone,clone3,futex \
    ./thread-demo
```

환경에 따라 `clone()` 또는 `clone3()`가 사용될 수 있다. 이번 결과에서는 worker 두 개에 대응하는 성공한 생성 호출이 두 번 나타났다.

Thread를 만드는 clone flag에서는 주소 공간, file descriptor table, signal handler, thread group 등을 공유한다는 의미의 flag를 볼 수 있다.

```text
CLONE_VM
CLONE_FILES
CLONE_SIGHAND
CLONE_THREAD
```

Main이 worker 종료를 기다리는 동안에는 `futex` 대기가 나타났다.

```text
pthread_create × 2 → clone/clone3 × 2
pthread_join       → futex wait
worker 종료        → futex 대기 해제
```

`futex`는 계속 CPU를 사용하며 상태를 확인하는 busy waiting 대신, 조건이 충족될 때까지 thread를 효율적으로 재우고 깨우는 기반으로 사용된다.

## Error handling과 남은 TODO

`pthread_create()`가 중간에 실패할 수 있으므로 실제로 생성된 thread 수를 `create_cnt`로 기록하고 그 수만큼 join하도록 개선했다.

아직 다음 실패 경로는 TODO로 남겼다.

- 일부 worker만 생성되었다면 생성된 worker를 모두 join한 뒤 결과를 출력하지 않고 실패로 종료해야 한다.
- 어떤 `pthread_join()`이 실패해도 뒤에 남은 생성된 thread의 회수를 계속 시도해야 한다.
- 배열 크기와 반복문의 숫자 `2`를 하나의 thread count 상수로 통일할 수 있다.

정상 경로는 올바르게 동작하지만, long-running daemon에서는 일부 초기화 실패 시 이미 생성한 thread를 어떻게 종료하고 회수할지 반드시 설계해야 한다.

## Linux System Software 직무와의 연결

Linux daemon과 agent는 수집, network I/O, logging처럼 성격이 다른 작업을 동시에 수행하기 위해 thread를 사용할 수 있다.

`sysmon-agent`에서는 collector thread와 network thread가 shared metrics를 함께 사용하게 된다. Thread lifecycle과 join을 이해해야 정상 종료 시 모든 worker를 안전하게 회수할 수 있고, 다음 단계의 mutex를 이용해 shared state를 보호할 수 있다.

## 학습 정리

### Learned

- 같은 프로세스의 thread는 address space와 여러 process resource를 공유한다.
- 각 thread는 별도의 stack, register 상태, 실행 흐름과 TID를 가진다.
- `pthread_create()`는 성공 시 0, 실패 시 error number를 직접 반환한다.
- Start routine은 `void *worker(void *arg)` 형태다.
- Worker에 전달한 객체는 worker가 사용하는 동안 lifetime이 유지되어야 한다.
- `pthread_join()`은 thread를 종료시키는 것이 아니라 종료를 기다리고 회수한다.
- Worker 실행 및 출력 순서는 scheduler에 의해 달라질 수 있다.
- 서로 다른 구조체 원소에 쓰고 join 후 읽는 현재 구조에는 data race가 없다.
- pthread는 Linux에서 clone과 futex 같은 kernel mechanism 위에 구현된다.

### APIs

- `pthread_create()`
- `pthread_join()`
- `strerror()`
- `sleep()`

### Commands

- `man 3 pthread_create`
- `man 3 pthread_join`
- `gcc -Wall -Wextra -Wpedantic -g -pthread`
- `ps -T -p <PID>`
- `ls /proc/<PID>/task`
- `grep '^Threads:' /proc/<PID>/status`
- `strace -f -e trace=clone,clone3,futex`

### Mistakes

- `struct info`를 typedef 이름인 `info`처럼 cast해 compile error가 발생했다.
- `strerror(err)`만 호출하면 message가 출력된다고 생각했지만, 이 함수는 문자열 pointer를 반환할 뿐이다.
- `pthread_join()`의 첫 번째 인자가 thread를 종료시키는 대상이라고 생각했지만 실제로는 종료를 기다릴 대상이다.
- `pthread_join()`의 `retval`을 process의 정수 exit status와 같은 개념으로 생각했다.
- Main이 join하지 않고 반환하면 worker까지 포함한 process 전체가 종료될 수 있다.

### Next

- 여러 thread가 같은 counter를 수정해 data race와 lost update를 관찰한다.
- `pthread_mutex_t`로 shared state를 보호한다.
- Mutex lock 범위와 error handling을 연습한다.
- 이후 `sysmon-agent`의 collector와 network thread 구조에 적용한다.
