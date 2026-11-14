---
title: "6_transpose_bankConflict"
date: "2026-06-29T14:45:55+09:00"
lastmod: "2026-06-30T11:35:57+09:00"
draft: false
description: ""
categories: ["study", "멀티코어및GPU프로그래밍"]
tags: []
ShowToc: true
---

### Dynamic shared memory allocation

- shared memory의 크기를 dynamic하게 allocation하고 싶으면 `extern` 키워드를 쓰면 된다.
- 그게 다야...

### Multi-bank architecture

- 기본적으로 memory는 single port를 쓰기 때문에 한번에 하나의 thread 만이 접근할 수 있다. 
- 그렇다면 threadblock 내의 thread들은 shared memory를 어떻게 쓰는 것일까?
- shared memory는 이를 위해 32개의 single port memory로 이루어져 있다. 이 하나하나의 조각들을 bank 라고 한다.
- bank 하나는 4B를 커버할 수 있다. (float 하나)

### Bank Conflicts

- thread들이 같은 bank에 접근하는 현상을 bank conflict라고 한다. 
- 이게 일어나면 해당 실행은 병렬로 처리할 수 없다.
- padding + coalescing으로 문제를 해결할 수 있다.
