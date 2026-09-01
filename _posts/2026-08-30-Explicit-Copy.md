---
layout: post
published: true
title: Explicit를 활용해 직접 초기화만을 가능 하게하기!
date: 2026-08-30 23:12:00 +0900
description: const
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
Cpp에서 복사초기화와 직접초기화중, 복사초기화를 막아보자!

#### explicit
암시적 변환을 막는 방법으로써 `explicit`키워드를 활용한다
 
##### Code 예시
``` cpp
#include <iostream>
using namespace std;

class Pius {
public:
	Pius() {}
	Pius(int energy) :_energy(energy) {}
	explicit Pius(const Pius& other) { _energy = other._energy; }
	int _energy = 100;

};

int main()
{
	Pius p1;
	p1._energy = 20;

	Pius p2(p1); // 직접 초기화 (direect-initialization) 가능
	Pius p3 = p2; // 복사 초기화 (Copy - Initialization) 불가
	return 0;
}
```



