---
layout: post
published: false
title: functor - 함수객체
date: 2026-09-30 21:13:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
함수 포인터는 변수를 포인터로 받는 일반 포인터에서 더 나아가 함수를 주소로 갖는 것이다.


---

stl은 잘 안써서 그런지 되게 잘 까먹는것 같아서 정리해두려 한다.

#### 타입만 포함:

``` cpp

void print()
{
	cout<<"Thank you AI"<<endl;
}
int main()
{
	using FuncPTR = void();
	FuncPTR* ptr = &print; // &:명시적 참조, 안적으면 암묵적 참조로됨
	
	ptr();
}

```

함수 타입만 함수 포인터로 지정하는 방식

#### 포인터까지 타입에 포함:

``` cpp

void print()
{
	cout<<"Thank you AI"<<endl;
}
int main()
{
	using FuncPTR2 = void(*)();
	FuncPTR ptr2 = &print; 
	
	ptr2();
}

```

DirectX에서도 이렇게 활용했었음

``` cpp
// 1. 함수 정의
LRESULT CALLBack WndProc(HWND hwnd, UINT msg, WPARAM wParam, LParam, lParam){} //이 함수를 등록


// 2. 함수 포인터 타입
using WndProcType = LRESULT(CALLBACK*)(HWND, UINT, WPARAM, LPARAM);

// 3. 등록
WNDCLASS wc = {};
wc.lpfnWndProc = &WndProc;; //함수 포인터로 등록 - 명시적 주소연산자
```