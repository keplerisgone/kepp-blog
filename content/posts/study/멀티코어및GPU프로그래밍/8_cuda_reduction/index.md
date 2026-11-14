---
title: "8_cuda_reduction"
date: "2026-06-29T14:45:55+09:00"
lastmod: "2026-07-12T12:53:51+09:00"
draft: false
description: ""
categories: ["study", "멀티코어및GPU프로그래밍"]
tags: []
ShowToc: true
---

sum을 계산하는 과정을 7단계로 최적화해보자.

### Parallel Reduction

![Image.png](Image-(8).png)

- 일단 계획은 위와 같이 tree-based로 병렬화를 하는 것임.
- 하지만 각 단계마다 sync가 필요한데, 이건 threadblock 안에서만 적용되고 global (warp)에서는 적용이 되지 않음.

![Image.png](Image-(9).png)

- 해결법: 각 커널이 각각의 단계를 담당하도록 하면 된다. 커널의 시작점이 sync 지점이 되기 때문. (자동으로 이게 된다고 한다) 이를 recursive kernels라고 한다. 중간에 smem을 쓴다는 점도 잊지 말자.
- mega kernel을 쓰는 방법도 있긴 하다.
1. Interleaved Addressing (power of 2)

![Image.png](Image-(10).png)

- 위와 같이 2의 거듭제곱 형태로 연산하는 방식이다.
- 하지만 각 단계마다 노는 thread가 많다는 점, memory 사용이 비효율적이라는 점(필요한 sum은 하나인데 불필요한 메모리 사용이 너무 많음)이 문제가 된다.
2. Interleaved Addressing

![Image.png](Image-(11).png)

- 그래서 tid를 연속된 수로 바꿔보았다.
- 하지만 bank conflict 문제가 발생한다. 특히 간격이 넓어질수록 conflict가 발생할 가능성이 커진다.
3. Sequential Addressing

![Image.png](Image-(12).png)

- 그래서 이번엔 주소값 또한 연속으로 지정해주었다.
- 하지만 아직도 노는 thread가 있다는 점이 걸린다.
