---
layout: post
published: false
title: Vector<T*> vs Vector<T>* (Stack을 통한 이해)
date: 2026-09-16 19:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
`Vector<T*>`와 `Vector<T>*`를 자유자재로 쓰지 못하는 느낌이 든다.

물론 전자는 T 포인터들을 담은 Vector Container이고,  
후자는 `Vector<T>`를 메모리 일렬로 가진 포인터

### Vector<T*>
포인터들을 담는 벡터로, 원소 하나하나가 포인터이다.

``` cpp
vector<int*> _vec;

int a =1, b=2;
_vec.push_bach(&a);
_vec.push_back(&b);

// _vec = [&a, &b];
```

```
_vec (stack/멤버)
┌──────────────┐
│ 내부 힙 버퍼  ─┼──▶ [ int* ][ int* ][ int* ] ...
└──────────────┘        │       │
                        ▼       ▼
                       int     int   (각 포인터가 가리키는 실제 대상)
```
- heap이 아닌 stack에 `_vec`이 들어간다
- `_vec`은 객체이므로 nullptr 불가!
- `_vec[0] = nullptr`가능
- **주의** - 소유권 문제 발생 : `new`로 넣었다면 vector 소멸되더라도 **Pointer가 가리키는 대상은 자동으로 delete 안됨**. `unique_ptr` / `shared_ptr`활용!

##### 해제시?
``` cpp
for(int i = 0 ; i <_vec.size() ; i <++)
	delete _vec[i];
```

##### 초기화
```cpp
vector<int*> v(5, nullptr); //원소 5개, nullptr
```

---

### `Vector<T>*`
벡터를 가리키는 포인터
```cpp
vector<int>* _vec = nullptr;
_vec = new vector<int>();
_vec->push_back(1);
(*_vec)[0];

delete _vec;
```

```
_vec (포인터)
┌────┐
│ 주소├──▶ vector<int> 객체 ──▶ [ int ][ int ][ int ] ...
└────┘
```

#### Stack cpp - with `Vector<T>* `
``` cpp
#pragma once
#include <iostream>
using namespace std;
#include "Vector.h"

template <typename T>
class Stack {

public:
	explicit Stack(int capacity) :_Vector(new Vector<T>(capacity)) {}
	~Stack() {}

	void push(const T& data) {
		_Vector->push_back(data);
	}
	T& top() {
		_Vector[_Vector->Size()];
	}
	void pop()
	{
		_Vector->resize(_Vector->Size() - 1);
	}
	void print()
	{
		int size = _Vector->Size();
		for (int i = 0; i < size; i++)
		{
			cout << (*_Vector)[size - i-1] << endl;
		}
		/*_Vector->Print();*/
	}
public:
	Vector<T>* _Vector; //Vector <T*> _Vector로했을떄도 확인해보기!

	int _capacity = 0;
};
```

### Stack cpp - with Vector<T*>
``` cpp
#pragma once
#include <iostream>
using namespace std;
#include "Vector.h"

template <typename T>
class Stack {

public:
	explicit Stack(int capacity):_vec(capacity)
	{
		
	}
	~Stack()
	{
		for (int i = 0; i < _vec.Size(); i++)
		{
			delete _vec[i];
		}
	}
	void push(const T& data)
	{
		/**_vec[_size] = data;
		_size++;*/
		_vec.push_back(new T(data));
	}
	T& Top() { return *(_vec[_vec.Size()-1]); }
	void Pop() {
		int last = _vec.Size() - 1;
		delete _vec[last];
		_vec.resize(last);
	}
	void Print()
	{
		int size = _vec.Size();
		for (int i = 0; i <size; i++)
		{
			cout << *(_vec[size - i-1]) << endl;
		}
	}
private:
	Vector<T*> _vec;

};

```