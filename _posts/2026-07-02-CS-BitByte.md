---
layout: post
published: true
title: Bit와 Byte
date: 2026-07-02 16:10:00 +0900
description: Memory 영역
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
Bit와 Byte에 대해서 알아보자

볼때마다 조금 헷갈렸던 Bite와 Byte, `1Byte = 8Bit`라는걸 매번 계산해서

정리가 필요하다 생각했다.

| 구분   | 의미                      | 표기      |
| ---- | ----------------------- | ------- |
| Bit  | 컴퓨터가 다루는 최소 단위 (0 또는 1) | b (소문자) |
| Byte | 8개의 Bit를 묶은 단위          | B (대문자) |

> 헷갈리는 포인트: `Bit`는 소문자 `b`, `Byte`는 대문자 `B`로 구분해서 쓰는 경우가 많다.
> 
> (예: `Mbps` vs `MB/s`)

**1 Byte = 8 Bit**

이 관계 하나만 기억하면, 아래 64bit / 32bit 시스템의 포인터 크기 계산이 훨씬 쉬워진다.

```
[     1byte     ] [    ] [    ] [    ] [    ] [    ] [    ] [    ]  : 8byte, 64bit
        ↑
[][][][][][][][]  : 8bit
```
한 1byte **덩어리**가 8bit임!

### 포인터에서의 주소 크기

###### 64bit와 32 bit에서의 차이

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/ComputerScience/PointerAdress64bit.png" alt="PointerAdress64bit" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> 64bit 메모리 주소값</td> <td width="42%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/ComputerScience/PointerAdress32bit.png" alt="PointerAdress32bit" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong>32bit 메모리 주소값</td> </tr> </table>
- 64Bit System : 메모리 주소를 64Bit로 표현
- 64Bit = 8byte (64 ÷ 8 = 8)
- 포인터는 **메모리 주소값**을 저장하는 변수이므로, 그 주소 표현하는 비트 수만큼의 크기

- 32Bit System : 메모리 주소를 32Bit로 표현


| 시스템  | 주소 공간 | 포인터 크기 | 16진수 표현 자리 수                   |
| ---- | ----- | ------ | ------------------------------ |
| 32비트 | 2^32  | 4바이트   | 8자리 (예: `0x008ff7f0`)          |
| 64비트 | 2^64  | 8바이트   | 16자리 (예: `0xf1bf97a900000000`) |


#### 결론
포인터 크기 = CPU/OS가 사용하는 주소공간의 비트수 / 8

- 64bit 시스템 -> 64 / 8 = 8Byte
- 32bit 시스템 -> 32/ 8 = 4Byte