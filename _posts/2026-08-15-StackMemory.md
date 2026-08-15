---
layout: post
published: true
title: 지역 변수의 참조 리턴하면 안되는 이유
date: 2026-08-15 12:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
결론부터 알아보자면 Memory에 있어서 StackMemory의 역할때문이다.  
Stack Memory는 수명이 함수스코프내부에 있을때이다.


#### 위험한 경우

##### Dangling Reference 
``` cpp
int& pius()
{
	int localVariable = 20;
	return localVariable; //컴파일 되지만 Dangling 위험!
}
```
지역 변수를 할당해놓고 그 지역변수의 참조를 반환한다면,  
함수가 리턴하는 순간 Stack Frame은 해제가 되면서  
참조는 유효하지 않은 메모리를 가리키게 되며, **Dangling Ref**를 가리키게 된다.  
이미 **죽은 메모리**를 가리키게 된다

#### 안전한 경우

##### 001 멤버 변수의 Reference 반환
```cpp
Class Pius()
{
private:
	int ClassVariable;
	
public:
	int& GetVariable(){ return ClassVariable; } //this가 살아있는한 가능

};
```

##### 002 Parameter 함수로 멤버 변수의 Reference 반환
``` cpp
template <typename T>
T& operator=(const T& other)
{
	return *this; //*this는 함수 호출 이전부터 존재하기에
}
```

003 Static / 전역 변수 참조
004 Container 요소 참조

##### Dx에서의 Out-parameter 패턴
``` cpp
void CreatSTH(ID3D11Device* device, ComPtr<ID3DBuffer>& outBuffer)
{
	//out buffer에 결과 채우기위함
}
```
반환의 느낌보다 함수가 **호출자가 소유한 객체**를 수정!

