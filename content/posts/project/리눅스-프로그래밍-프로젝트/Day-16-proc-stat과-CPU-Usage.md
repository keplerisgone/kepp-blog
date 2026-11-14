---
title: "Day 16: /proc/stat과 CPU Usage"
date: "2026-09-01T14:25:14+09:00"
lastmod: "2026-09-01T14:25:14+09:00"
draft: false
description: ""
categories: ["project", "리눅스-프로그래밍-프로젝트"]
tags: []
url: "/posts/project/day-16-proc-stat과-cpu-usage/"
weight: 16
ShowToc: true
---

Linux의 `/proc/stat`에서 전체 CPU 누적 시간을 두 번 읽고, 두 snapshot의 차이로 CPU usage를 계산하는 collector를 구현했다.

## /proc/stat의 CPU 시간

첫 번째 `cpu` 행은 모든 CPU core를 합산한 누적 시간이다.

```text
cpu user nice system idle iowait irq softirq steal guest guest_nice
```

각 숫자는 순간 사용률이 아니라 시스템 부팅 이후 해당 상태에서 소비한 누적 CPU 시간이다. `guest`와 `guest_nice`는 이미 `user`와 `nice`에 포함되므로 total에 다시 더하지 않았다.

```text
idle_all = idle + iowait

non_idle =
    user + nice + system
    + irq + softirq + steal

total = idle_all + non_idle
```

## 첫 행 읽기와 파싱

`fopen()`으로 `/proc/stat`을 열고 `fgets()`로 첫 행을 문자열 buffer에 읽었다. `sscanf()`로 label과 8개의 누적값을 구조체에 저장했다.

```text
%3s  → 최대 3글자의 label
%llu → unsigned long long 값
```

Label 한 개와 숫자 8개를 읽으므로 `sscanf()` 반환값이 정확히 9인지 검사하고, `strcmp()`로 label이 `cpu`인지 확인했다.

누적값에는 `unsigned long long`을 사용했다. `sscanf()`에는 숫자를 저장할 구조체 멤버의 주소를 전달했다.

```text
label          → char 배열 자체를 전달
&times->user   → 숫자를 저장할 멤버의 주소
```

## Snapshot과 CPU usage

한 번 읽은 누적값만으로는 현재 사용률을 알 수 없다. 두 snapshot 사이에 3초 간격을 두고 차이를 계산했다.

```text
before snapshot
→ sleep(3)
→ after snapshot
```

```text
delta_total = after_total - before_total
delta_idle  = after_idle - before_idle

usage =
    (delta_total - delta_idle)
    / delta_total
    × 100
```

정수 나눗셈으로 소수점이 사라지지 않도록 `double`로 변환했다. 실제 실행에서는 다음과 같은 결과를 확인했다.

```text
CPU usage: 0.54%
```

환경의 실행 부하에 따라 결과는 매번 달라진다.

## 방어 검사

다음 경우를 오류로 처리했다.

- `/proc/stat` open 또는 read 실패
- 예상하지 못한 EOF
- `sscanf()`가 9개 필드를 모두 변환하지 못함
- 첫 label이 `cpu`가 아님
- `fclose()` 실패
- 누적 counter가 이전 snapshot보다 작아짐
- `delta_total == 0`
- `delta_idle > delta_total`

`read_cpu_times()`는 기본 status를 실패로 두고, 파싱에 성공한 경우에만 0으로 바꿨다. `fopen()` 이후의 모든 경로는 공통 `cleanup` label에서 `fclose()`를 시도한다.

## strace 관찰

다음 system call을 중심으로 실행 흐름을 확인했다.

```bash
strace -yy \
  -e trace=openat,read,close,clock_nanosleep \
  ./proc/cpu-usage
```

관찰한 lifecycle은 다음과 같다.

```text
openat("/proc/stat")
→ read()
→ close()
→ clock_nanosleep()
→ openat("/proc/stat")
→ read()
→ close()
→ CPU usage 출력
```

Source에서는 `sleep(3)`을 사용했지만 libc를 거친 실제 system call은 `clock_nanosleep()`으로 관찰될 수 있다. Program 시작 시 보이는 `/etc/ld.so.cache`와 `libc.so.6` 접근은 dynamic linker가 공유 라이브러리를 읽는 과정이다.

## Linux System Software 직무와의 연결

`sysmon-agent`의 CPU collector는 같은 원리로 `/proc/stat`을 주기적으로 읽는다. Metric은 단일 누적값이 아니라 이전 snapshot과 현재 snapshot의 차이를 이용해 계산해야 하며, collector는 이전 상태를 다음 주기까지 유지해야 한다.

## Learned

- `/proc/stat`의 CPU 값은 부팅 후 누적 시간이다.
- CPU usage를 구하려면 시간 간격을 둔 두 snapshot이 필요하다.
- `guest`와 `guest_nice`를 다시 더하면 CPU 시간을 중복 계산한다.
- `sscanf()` 반환값으로 실제 변환된 필드 수를 검사해야 한다.
- `/proc` 파일도 open, read, close의 resource lifecycle을 가진다.
- 누적 counter의 역행과 0인 delta를 검사해야 안전하게 계산할 수 있다.

## APIs

- `fopen()`
- `fgets()`
- `sscanf()`
- `strcmp()`
- `ferror()`
- `fclose()`
- `sleep()`

## Commands

- `head -n 1 /proc/stat`
- `watch -n 1 'head -n 1 /proc/stat'`
- `strace -yy -e trace=openat,read,close,clock_nanosleep`

## Mistakes

- `fgets()`로 읽은 변수는 `buffer`인데 처음 `sscanf()`에 존재하지 않는 `line`을 전달했다.
- 성공 경로에서 `read_cpu_times()`의 반환값을 처음에는 작성하지 않았다.
- 오류 경로마다 `fclose()`와 `return`이 반복되어 공통 cleanup 구조로 정리했다.

## TODO

- 측정 간격을 상수 또는 인자로 분리
- CPU 계산 로직을 별도 함수로 분리
- 여러 core의 `cpu0`, `cpu1` 행별 usage 계산은 필요할 때 확장

## Next

- `/proc/meminfo`를 읽고 memory usage를 계산한다.
