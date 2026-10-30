---
layout: post
published: true
title: Iterator의 구현과 이해
date: 2026-10-30 09:03:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---

[Vector](https://piusai.github.io/c++/2026/09/16/Container.html)와 같은 Container에서 Traversal(순회)하기 위해서 많이 활용되는 Iterator,  
이번 글에서는 Iterator를 직접 구현해보겠다.

iterator를 지원하려면 각 클래스가 다음을 갖춰야한다.
- Iterator class : 
  - `operator*`: 역참조
  - `operator++()` : 전위
  - `operator++(int)` : 후위 
  - `operator+(int c)` : 건너뜀
  - `operator==`, `operator!=` : for loop종료 조건에서의 필수!

- Container(Vector): `begin(), end()`
### iterator, 직접 구현

``` cpp
#pragma once
#include <iostream>
#include <assert.h>
using namespace std;

template <typename T>
class Iterator
{
public:
	Iterator():_ptr(nullptr) {}
	Iterator(T* ptr): _ptr(ptr) {}
	//소유권 없는 wrapper여서 소멸자 x!

	T& operator*() { return *_ptr; }

	//전위
	Iterator& operator++() {
		++_ptr;
		return *this;
	}

	//후위
	Iterator operator++(int) {
		Iterator temp = *this;
		++_ptr;
		return temp;
	}


	Iterator operator+(int counter) const
	{
		Iterator temp = *this;
		temp._ptr = temp._ptr + counter;
		return temp;
		//return Iterator(_ptr +counter) 이렇게 처리해도됨
	}

	//읽기만 하기에, 객체 자신(this)를 수정 않겠다!!
	bool operator==(const Iterator& other) const { return other._ptr == _ptr; }
	bool operator!=(const Iterator& other) const { return other._ptr != _ptr; }

public:
	T* _ptr = nullptr;
};

template <typename T>
class Vector {
	

public :
	using iterator = Iterator<T>;
	iterator begin() { return iterator(_buffer); }  //Iterator(T*) 생성자로 간다
	iterator end() { return begin() + _size; }

	explicit Vector(int capacity) : _capacity(capacity), _buffer(new T[capacity]) {}
	~Vector() {
		delete[] _buffer;
	}
	T& operator[](int index)
	{
		assert(index < _size && index >= 0);
		return _buffer[index];
	}
	void push_back(const T& data)
	{
		if (_capacity == _size) {
			int newcapacity = _capacity * 1.5;
			if (newcapacity <=_capacity) newcapacity = _capacity + 1;
			reserve(newcapacity);
		}
		_buffer[_size] = data;
		_size++;
	}
	void pop_back() {
		if(_size==0) return;
		//_buffer[_size] = nullptr; -> T가 int라면 컴파일 error / 필요 없음
		_size--;
	}
	void reserve(int newcapacity)
	{
		if (newcapacity <= _capacity) return;
		T* newbuffer = new T[newcapacity];
		for (int i = 0; i < _size; i++)
		{
			newbuffer[i] = _buffer[i];
		}
		delete[] _buffer;
		_buffer = newbuffer;
		_capacity = newcapacity;
	}
	void Print()
	{
		for (int i = 0; i < _size; i++)
		{
			cout<<_buffer[i]<<endl;
		}
	}

	int Capacity() { return _capacity; }
	int Size() { return _size; }


public:
	T* _buffer=nullptr;
	int _capacity=0;
	int _size =0;
};
// vector 복사생성자 / 대입 연산자 없어서 Vector 복사하면 double free날 수있음
// 

int main()
{
	Vector<int> vt(2);
	vt.push_back(20);
	vt.push_back(60);
	vt.push_back(70);
	vt.push_back(30);
	vt.Print();

	Vector<int>::iterator it;
	for (it = vt.begin(); it != vt.end(); it++)
	{
		cout << *it<<endl;
	}

	return 0;
}
```

- Iterator는 Vector의 Wrapper가 아닌, `T*`원소 포인터의 Wrapper!

- iterator의 `_ptr`은 현재 가리키는 원소 위치를 들고 있다.
- Vector의 `_buffer`는 그 배열 시작주소를 들고있다. `new[]/delete[]`로 메모리 관리


- `begin()`으로 `Iterator(T*)` 생성자 호출할때, `_buffer`의 주소를 복사해서 `_ptr`에 넣기 때문에 같은 주소를 가리킴.  
  -> 시작 순간(`begin()`)에는 같은 주소를 가리킨다.
- 그 이후 iterator는 ++로 `_ptr`을 옮긴다 - 이때부터 `_buffer` != `_ptr`
- `end()`는 마지막 원소 다음위치 이기 때문에 `begin() + _size`이다.

- `Iterator`에 소멸자를 두지 않는 이유는 소유권이 없기 때문이고, 복사를 자유롭게 하기위해서이다.
	→ `Iterator temp = *this;`나 값으로 반환할떄 복사가 일어나는데 `delete[]`를 하면 복사본마다 같은 메모리를 해제 하게 된다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/cpp/iteratorPointer.png" alt="UML" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>iteratorPointer</strong> </td>  </tr> </table>

---
#### `const`의 추가 이해

``` cpp
bool operator ==(const Iterator& other) const
//               ^^^^^ ①                ^^^^^ ②
```

- ① 인자 쪽 `const Iterator& other` : 비교 대상(`other`)을 수정하지 않겠다는 약속
  : `const&`여야 `vt.end()`같은 임시 객체도 인자로 받을 수 있음
- ② 뒤쪽 `const` : 호출한 객체 자신(`this`)을 수정하지 않겠다는 약속

 `it`이 const가 아닌 일반 객체이기 때문에 현재 코드에서는 컴파일이 되지만
 아래와 같이 왼쪽 피연산자가 const인 순간 에러가 난다.

``` cpp
void f(const Iterator<int>& a, const Iterator<int>& b)
{
	if( a==b ){} //② 가 없으면 컴파일 에러!
}
```

- const 객체에서 non-const 멤버 호출은 불가능하다!
- 읽기만 하는 함수는 습관적으로 `const`를 붙이기!