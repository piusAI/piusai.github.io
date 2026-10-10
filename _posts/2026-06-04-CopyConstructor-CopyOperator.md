---
layout: post
title: 복사생성자와 복사 대입 연산자의 명시적 차단
date: 2026-06-04 11:40:00 +0900
description: 리소스 이중 해제(Dangling Pointer)를 방지하기 위해 복사 생태계를 제어하는 컴파일 타임 설계 전략을 알아봅니다.
categories:
  - C++
tags:
  - DirectX
  - Framework
  - Architecture
  - Graphics-Pipeline
---
복사생성자만을 명시적으로 막아봅시다.


## 서론 : 왜 복사를 강제로 막아야 하는가?
억지로 막아줘야하는 경우들이 있다.
이러한 패턴들을 클래스 인터페이스 구현할때 많이 이용한다.
OOP의 장점은 코드를 다른사람들도 뜯어 활용하기 위함에 있는데,

`const [class]& other`도 그 예시이다.
구현부에서 개발자가 에러를 뱉도록 만들어놓은 좋은 설계이다.



## 실험 예시 코드

{% highlight cpp %}
#include <iostream>
using namespace std;


class DonOverride
{
public:
	DonOverride() {}
	explicit DonOverride(const DonOverride& other): data(other.data) { } // 복사 생성자
	~DonOverride() {}

	DonOverride& operator=(const DonOverride& other) = delete; // 복사 대입 연산자(overloading)


public:
	int data = 0;
	
};
int main()
{
	DonOverride D1;
	DonOverride D2(D1); //복사 생성자

	DonOverride D3 = D1; //explicit가 막아줌
	DonOverride D4;
	D4 = D1; //delete가 복사 대입 연산자 막아줌

	// 포인터는 당연히 메모리 주소값일 뿐이어서 오버로딩 호출 안됨!
	// 얕은 복사 밖에 안되겠다!
	DonOverride* ptr_D1 = new DonOverride();
	
	DonOverride* ptr_D2(ptr_D1);
	DonOverride* ptr_D3;
	ptr_D3 = ptr_D1;

	delete ptr_D1;
	/*
	delete D2;
	delete D3;
	*/
	// 두번 메모리 해제하면 Dangling Pointer!

	cout << "Done" << endl;
	return 0;
}
{% endhighlight %}




복사 생성자는 당연히 가능하다

#### 정상 작동하는 복사 생성자

{% highlight cpp %}

//DonOverride class Scope
explicit DonOverride(const DonOverride& other){}


//main Scope
DonOverride D1;
DonOverride D2(D1);

{% endhighlight %}



다른 코드들은 거들 뿐이다.


* 01 암시적 복사 변환 막기
* 02 암시적 operator= 막기

#### 암시적 복사 변환 막기 : explicit

{% highlight cpp %}
// DonOverride Class Scope
explicit DonOverride(const DonOverride& other){}


// main Scope
DonOverride D3 = D1; //컴파일 에러

{% endhighlight %}




#### 암시적 operator= 막기 : delete활용



{% highlight cpp %}

//DonOverride class Scope
DonOverride& operator=(const DonOverride& other) = delete;


//main Scope
DonOverride D4;
D4 = D1; //컴파일 에러
{% endhighlight %}



이렇게 컴파일러가 기본 `operator=`를 자동 생성해주는 것을 막는다.