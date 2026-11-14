---
title: "7_cuda_dnn"
date: "2026-06-29T14:45:55+09:00"
lastmod: "2026-07-12T12:53:51+09:00"
draft: false
description: ""
categories: ["study", "멀티코어및GPU프로그래밍"]
tags: []
ShowToc: true
---

### Deep neural network

- input이 들어오면 weight를 곱하고 bias를 더해 output를 만들어내는 unit으로 이루어진 거대한 신경망.
- 각 layer는 종류에 따라 fully connected, sparsely connected layer로 나뉠 수 있다.

### 2D Convolution

![Image.png](Image-(5).png)

- 각 image는 pixel들을 특정 값으로 변환한 형태로 나타낼 수 있다.
- 이미지에 filter mask (혹은 filter, kernel)를 convolution 연산을 하면 이미지의 특정한 '특징'을 뽑아낼 수 있다.
- DNN layer에서는 N개의 filter를 사용해 N개의 feature map(response map)를 뽑아내는 형식이다.

### Activation function

- 비선형성을 추가하기 위해 layer 사이에 들어가는 함수이다.
- ReLU, sigmoid 등이 있다.

### Convolution layer

- Convolution layer의 계산 과정을 코드로 러프하게 나타내면 다음과 같다.

![Image.png](Image-(6).png)

- 즉 convolution은 7중 for문으로 이루어져 있는 것!
- 병렬화는 작은 수준의 병렬화만 가능한 n,m,h,w 쪽보다 c,p,q 쪽을 병렬화하는 것이 효율이 좋다. (왜지)
- 또한 convolution 중간에 생기는 중복된 원소의 사용은 역시 shared memory의 사용으로 해결한다.

### im2col

- 아무래도 GPU는 matmul에 특화되어 있다. Convolution 연산을 이것으로 바꾸는 방법은 없을까?
- **im2col은** Convolution 연산을 matmul로 간편하게 만들어 주는 단계이다.

![Image.png](Image-(7).png)

- weight와 input을 펼쳐 matrix를 만들고, 이를 matmul해 output matrix를 만들어내는 방식이다. 
- 이외에도 convolution을 수행하는 다른 방법으로는 FFT-based convolution, [Winograd transform](craftdocs://open?blockId=58B8B8FF-8B37-4DF6-862A-A4DB0D8C4BDA&spaceId=13dc16b5-1640-e515-ca68-8cd96caa1139) 등이 있다.
