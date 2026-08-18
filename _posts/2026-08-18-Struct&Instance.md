---
layout: post
published: true
title: Struct는 메모리 어떻게 차지하길래?
date: 2026-08-18 12:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
Struct를 전역에서 instance로써 설정하려하면 오류를 뱉는다  
왜그럴까..?  

#### Struct 중요!

##### 01 구조체(Struct)자체는 메모리를 차지 하지 않는다.

``` cpp
struct StatStruct
{
	int Messi;
	int Ronaldo;
	int Kane;
};
```
이런 모양의 data를 묶은 설계도일 뿐,  
실제 ***메모리***가 할당되지 않았다.

``` cpp
StatStruct SoccerStat;

SoccerStat.Messi = 10;
SoccerStat.Ronaldo = 7;
// 전역 스코프에 있으면 컴파일 에러!

int main()
{
...
}
```

전역 Scope(함수 밖)에는 오직 **선언문, 초기화가 포함된 정의**만이 올 수있다

#### 해결 방법

##### 1) 선언과 동시에 초기화
``` cpp
StatStruct SoccerStat = {10, 7, 9};
```

##### 2) Main()등 함수 안에서 값 대입

``` cpp
StatStruct SoccerStat
int main()
{
	SoccerStat.Messi = 10;
	SoccerStat.Ronaldo = 7;	
	SoccerStat.Kane = 9;
}
```
##### 3) 초기화 함수를 따로 만들기
``` cpp
StatStruct SoccerStat
void InitSoccerStat()
{
	SoccerStat.Messi = 10;
	SoccerStat.Ronaldo = 7;	
	SoccerStat.Kane = 9;
}
int main()
{
	InitSoccerStat(;)
}
```


#### 정리
Struct 정의는 Compile Time에만 존재하는 타입정보,
Struct **변수(instance)** 는 선언되는 순간 메모리를 차지!
변수 **대입**하는 코드는 반드시 ***함수안 / 초기화 리스트*** 에 있어야함!