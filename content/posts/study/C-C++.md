---
title: "C/C++"
date: "2026-06-29T14:45:55+09:00"
lastmod: "2026-06-30T11:35:57+09:00"
draft: false
description: ""
categories: ["study"]
tags: []
ShowToc: true
---

## current

일주일동안 공부하고 임베디드 리눅스로 넘어가기 프로젝트

| **날짜** | **목표**          | **해야 할 것**                                        |
| ------ | --------------- | ------------------------------------------------- |
| 1일차    | C 실행 환경과 문법 최소화 | `gcc`, `clang`, `make`, 변수, 조건문, 반복문, 함수          |
| 2일차    | 포인터/배열          | `int *p`, 배열과 포인터 차이, 문자열, `sizeof`               |
| 3일차    | 메모리와 struct     | `malloc/free`, `struct`, `typedef`, 헤더 분리         |
| 4일차    | 비트 연산           | `&`, `                                            |
| 5일차    | Linux 파일 I/O    | `open`, `read`, `write`, `close`, file descriptor |
| 6일차    | 프로세스/시스템 호출 맛보기 | `fork`, `exec`, `pipe`, `signal` 개념만              |
| 7일차    | 작은 프로그램 하나 만들기  | `/proc`, `/sys`, 파일 입출력, Makefile 포함              |

**목표: Linux에서 C로 “장치 파일처럼 보이는 것”을 읽고 쓰는 프로그램 만들기**

[Beej’s Guide to C Programming](https://beej.us/guide/bgc/) 일부만 보기
→ 포인터, struct, 파일 I/O 실습
→ [Beej’s Network Programming](https://beej.us/guide/bgnet) 앞부분으로 socket 감각 익히기
→ [Bootlin Embedded Linux](https://bootlin.com/docs/) 자료로 Buildroot/커널/디바이스 트리 입문

gcc 컴파일 옵션 공부

처음부터 “커널 개발자”를 목표로 잡으면 진입장벽이 너무 높습니다. 대신 **C + Linux system call + socket + mini-shell + tiny HTTP server + Yocto/Buildroot 맛보기**까지 만들고, 이후 BSP/드라이버로 들어가는 게 현실적입니다.

| Title | Items |
| ----- | ----- |
|       |       |
|       |       |
|       |       |
