---
title: "mnd: 터미널 마음챙김 기록 CLI"
date: "2026-08-17T12:53:33+09:00"
lastmod: "2026-08-17T12:53:33+09:00"
draft: false
description: ""
categories: ["project"]
tags: []
ShowToc: true
---

`mnd`는 하루 동안 떠오르는 생각, 감정, 신체감각과 판단을 빠르게 남기기 위해 C로 만든 작은 CLI다. 기록을 평가하거나 교정하지 않고, 사용자가 지정한 분류와 내용을 그대로 저장한다.

```sh
mnd "일반 기록"
mnd -t "생각"
mnd -e "감정"
mnd -b "신체감각"
mnd -j "판단"
mnd             # 오늘 기록 확인
mnd today       # 오늘 기록 확인
```

기록은 `~/.local/share/mnd/YYYY-MM-DD.log`에 사람이 직접 읽을 수 있는 형식으로 저장한다.

```text
09:12|note|첫 기록
09:18|thought|작은 부분부터 시작해보자
09:20|body|어깨가 뻐근하다
```

현재 버전은 표준 C11과 일반적인 Linux/POSIX API를 사용하며, 날짜별 텍스트 파일 기록과 오늘 기록 조회만 제공한다. TUI, SQLite, 네트워크, 동기화와 활동 시작·종료 기능은 실제로 필요해질 때 추가할 예정이다.
