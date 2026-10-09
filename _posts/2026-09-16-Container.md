---
layout: post
published: true
title: 기본 자료구조 (Vector, Stack, Queue)
date: 2026-09-16 19:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
STL에서 활용 법을 알아보고, 
Vector, Stack, Queue(원형)의 기본 자료구조를 구현한다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/StackQueue/Stack.png"  alt="Stack" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Stack</strong> </td> <td width="32.5%" style="text-align: center; border: none; padding: 3px;"> <img src="/assets/postimg/StackQueue/Queue.png" alt="Queue.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Queue</strong></td> </tr> </table>


### 01 Abstract

|        | Vector                 | Stack            | Queue (원형)                              |
| ------ | ---------------------- | ---------------- | --------------------------------------- |
| 종류     | 동적 배열 (구현체)            | ADT (LIFO)       | ADT (FIFO)                              |
| 꺼내는 순서 | 인덱스로 아무거나 가능           | 마지막에 넣은 것        | 처음에 넣은 것                                |
| 내부 저장소 | `T*` buffer            | `Vector<T*>`     | `Vector<T*>` 또는 `Vector<T>*`            |
| 넣기     | `push_back` 분할상환 O(1)  | `Push` 분할상환 O(1) | `Push` 분할상환 O(1)                        |
| 빼기     | `pop_back` O(1)        | `Pop` O(1)       | `Pop` O(1)                              |
| 접근     | `[i]` O(1)             | top만 O(1)        | front만 O(1)                             |
| 늘어날 때  | `reserve` (×1.5, O(n)) | Vector가 처리       | `Grow` (×1.5, O(n))                     |
| 핵심 변수  | `_size`, `_capacity`   | `_size`          | `_front`, `_back`, `_size`, `_capacity` |

### 02 Vector

#### 02-01 이중 벡터 표현하기
``` cpp
#include <vector>
using namespace std;

vector<vector<bool>> DoubleVectorBool(3, vector<bool>(3, false));
vector<vector<int>> DoubleVectorInt(4 ,vector<int>(4, -1));

```

- 초기화된 메모리 구조는 다음과 같다
```
//DoubleVectorBool - false 0으로 표현
{{ 0, 0, 0 },
{ 0, 0, 0 },
{ 0, 0, 0 }}

//DoubleVectorInt
{{-1, -1, -1, -1},
{-1, -1, -1, -1},
{-1, -1, -1, -1},
{-1, -1, -1, -1}}
```
- 접근 : `doubleVectorInt[Row][Column]` - `doubleVectorInt[1][2] `= 5;

### 직접 구현하는 Vector, Stack, Queue

#### 01. 직접 구현하는 Vector .cpp


``` cpp
#include <iostream>
#include <assert.h>
using namespace std;

template <typename T>
class Vector{
public:
	
	explicit Vector(int capacity) :_capacity(capacity), _buffer(new T[capacity])
	{
		assert(_capacity >= 0);
	}

	//깊은 복사 추가 (claude)
	Vector(const Vector& other) : _buffer(new T[other._capacity]), _size(other._size), _capacity(other._capacity)
	{

		for (int i = 0; i < other._capacity; i++)
		{
			_buffer[i] = other._buffer[i];
		}
	}
	
	~Vector()
	{
		delete[] _buffer;
	}

	//복사 연산자 추가 (claude)

	Vector& operator=(const Vector& other)
	{
		if (&other == this) return *this; //주소값으로 비교

		T* newBuffer = new T[other._capacity];
		for (int i = 0; i < other._size; i++)
		{
			newBuffer[i] = other._buffer[i];
		}
		delete[] _buffer;
		_buffer = newBuffer;
		_size = other._size;
		_capacity = other._capacity;
		return *this;
	}

	T& operator[](int index)
	{
		assert(index >= 0 && index < _size);
		return _buffer[index];
	}
	T* begin() { return _buffer; }
	T* end() { return _buffer + _size; }

	void pop_back()
	{
		assert(_size >0);
		--_size;
	}
	void push_back(const T& data)
	{
		if (_size == _capacity)
		{
			//이사해야함
			reserve();
		}
		_buffer[_size] = data;
		_size++;

	}
	void reserve(float size=1.5)
	{
		int newCapacity = _capacity * size;
		if (newCapacity <= _capacity) {
			newCapacity = _capacity + 1; //최소 1개 수정
		}
		if (newCapacity <= _capacity) return;
		T* newBuffer = new T[newCapacity];

		for (int i = 0; i < _size; i++)
		{
			newBuffer[i] = _buffer[i];
		}
		delete[] _buffer;
		_buffer = newBuffer;
		_capacity = newCapacity;
	}
	void resize(int newSize)
	{
		assert(newSize<=_capacity);
		_size = newSize;
	}
	void clear()
	{
		_size = 0;
	}
	void print()
	{
		for (int i = 0; i < _size; i++)
		{
			cout << _buffer[i]<<" ";
		}
	}


	int Size() { return _size; }
	int Capacity() { return _capacity; }

public:
	T* _buffer = nullptr;
	
private:
	int _size =0;
	int _capacity=0;

};

int main()
{
	Vector<int> vec = Vector<int>(2);
	vec.push_back(20);
	vec.push_back(90);
	vec.push_back(80);
	vec.push_back(40);

	//vec.print();

	for (auto& v : vec)
	{
		cout << v<< " ";
	}
	return 0;
}



```

복사 연산자, 복사 생성자는 claude와 함께 놓친부분 확인함
iterator를 활용하기위해 begin, end도 넣음


#### 2. 직접 구현하는 Stack (LIFO)

```
Push(10) → Push(20) → Push(30) → Pop()

       │ 30 │ ← top (Pop 대상)
       │ 20 │
       │ 10 │
       └────┘
```


#####  01 `Vector<T*>`를 활용한 Stack 구현

``` cpp
#include <iostream>
using namespace std;
#include "Vector.h"

template <typename T>
class Stack {
public:
	explicit Stack(int capacity):_vec(capacity), _capacity(capacity){}
	~Stack()
	{
		for (int i = 0; i < _size; i++)
			delete _vec[i];
	}
	Stack(const Stack&) = delete;                  // 막아두기
	Stack& operator =(const Stack&) = delete;      // 막아두기

	T* Back()
	{
		return _vec[_size - 1];
	}

	void Pop()
	{
		if (_size == 0) return;
		delete _vec[_size - 1];
		_vec.pop_back();
		--_size ;
	}

	void Push(const T& other)
	{
		_vec.push_back(new T(other));
		++_size ;
	}

	void Print()
	{
		for (int i = 0; i < _size; i++)
		{
			cout << *_vec[_size-i-1] << endl;
		}
	}

private:
	Vector<int*> _vec;
	int _size = 0;
	int _capacity = 0;
};
```


#### 3. 직접 구현하는 Queue(FIFO)

**동작**:
```
Push(A) → Push(B) → Push(C) → Pop()

나가는 쪽                        들어오는 쪽
   front ─→ [ A ][ B ][ C ]
					         ↑ back
            (Pop 대상)          (다음 Push 자리는 back이 가리키는 빈칸)
```

#####  01 `Vector<T>*`를 활용한 Queue 구현

``` cpp
template <typename T>
class Queue{

public:
	explicit Queue(int capacity) : _data(new Vector<T>(capacity)), _capacity(capacity)
	{
		_data->resize(capacity); // 원형 큐 다 채워주기!

	}
	~Queue()
	{
		delete _data;
	}

	void Push(const T& value)
	{
		//이사
		if (_QueueSize == _capacity)
		{
			int newcapacity = _capacity * 1.5;
			if (newcapacity <= 1) newcapacity++;
			
			//새로 원형 queue container 만들어주고!
			Vector<T>* newBuffer = new Vector<T>(newcapacity);
			_data->resize(newcapacity); //다 채워줘야함 원형 queue, 복사 loop보다 먼저
			for (int i = 0; i < _QueueSize; i++)
			{
				(*newBuffer)[i] = (*_data)[(i+_front)%_QueueSize];
			}
			delete _data;
			_data = newBuffer;
			
			_front = 0;
			_back = _QueueSize;
			_capacity = newcapacity;
		}
		_QueueSize++;
		(*_data)[_back] = value;
		_back = (_back +1) % _capacity;
		
	}
	

	T& Front() { return (*_data)[_front]; }
	T& Back() { return (*_data)[(_back-1+_capacity) % _capacity]; }
	void Pop()
	{
		if (_QueueSize == 0) return;
		_front = (_front + 1) % _capacity; //capacity 원형 Queue
		_QueueSize--;
	}
	void Print()
	{
		for (int i = 0; i < _QueueSize; i++)
		{
			cout<<(*_data)[(i + _front) % _capacity]<<endl;
		}
	}
private:
	Vector<T>* _data = nullptr;
	int _capacity = 0;

	int _front = 0;
	int _back = 0;

};

```

원형 Queue로 front / back을 사이클 돌림


#####  02 `Vector<T*>`를 활용한 Queue

``` cpp
template <typename T>
class Queue
{
public:
	explicit Queue(int capacity) :_vec(capacity), _capacity(capacity)
	{
		_vec.resize(_capacity); //
		//원형 큐 vector의 size는 capacity로!
		// ->queue의 _size 멤버변수랑 다름!
		for (int i = 0; i < capacity; i++)
		{
			_vec[i] = nullptr;
		}
	}

	~Queue()
	{
	
		for (int i = 0; i < _size; i++)
		{
			int idx = (_front + i) % _capacity;
			delete _vec[idx];
		}
	}

	void Push(const T& data)
	{

		if (_size == _capacity)
		{
			Grow();
		}

		_vec[_back] = new T(data);
		_back = (_back + 1) % _capacity; 
		++_size;
	}

	void Grow()
	{
		int newcapacity = (int)(_capacity * 1.5);
		if (newcapacity <= _capacity) newcapacity = _capacity+1;

		Vector<T*> _newVec(newcapacity);
		_newVec.resize(newcapacity);
		for (int i = 0; i < newcapacity; i++) _newVec[i] = nullptr;
		for (int i = 0; i < _size; i++)
		{
			_newVec[i] = _vec[(i + _front) % _capacity];
		}
		
		_vec = _newVec;
		_capacity = newcapacity;
		//여기서 만들어준 _newVec의 데이터를 지우면 안됨!! --- 아래 추가 설명
		_front = 0;
		_back = _size;
			
	}

	T* Front()
	{
		if (_size == 0) return nullptr;
		return _vec[_front];
	}

	void Pop()
	{
		if (_size==0) return;
		delete _vec[_front];
		_vec[_front] = nullptr;
		_front = (_front + 1 + _capacity) % _capacity;
		--_size;
	}

	void Print()
	{
		for (int i = 0; i < _size; i++)
		{
			cout<<*_vec[(i + _front+ _capacity) % _capacity]<<endl;
		}
	}


public:
	Vector<T*> _vec;
	int _capacity = 0;
	int _front = 0;
	int _back = 0;
	int _size = 0;
};

int main()
{
	Queue<int> qu(20);
	qu.Push(20);
	qu.Push(40);
	qu.Push(60);
	qu.Push(80);

	qu.Print();

	return 0;
}
```


#### 추가 헷갈릴만한 지점 / 컨셉
- 원형 Queue에서는 front로 first로 들어온 데이터를 가리키지만, `pop`으로 나가면서 데이터를 비워두기때문에 `++Front`로 하나를 더해줌
- `_capacity` : Vector buffer가 몇칸 실제로 갖고있는지,  원형 큐로 사이클을 `%`모듈러 계산으로 나머지로써 `front, back`이 가리키는걸 돌아가도록 함.
- `back`은 마지막 데이터를 가리키는것이 아니라 비어있는 `T*`를 가리키고있음!( `T* Vector<T>::end`와 유사)
- Queue에서의 Vector는 resize(capacity)로 `capacity ==size`임!
- Queue에서의 `_size` 멤버 변수와 `_vec.Size()`는 다름!!


#### 아래 추가 설명
- 왜 `_newvec`는 해제하면 안되는가?
`Vector<T*> _newvec`는 스택에 있는 객체이고 그 Vector객체 안에 T* 주소값들이 들어가있는데
`int`같은 데이터 힙에 `new T(data)`로 만들어져있고, `_newvec`과 `_vec`이 현재 같은 주소를 나누어서 가리키고 있는 상태

```
_newvec._buffer → [ p0 | p1 | p2 | null ]  ─┐
                                            ├─ 같은 주소 → 힙의 20, 40, 60
_vec._buffer    → [ p0 | p1 | p2 | null ]  ─┘   (operator= 이후)
```

-> `delete _newvec[i]`를 하면 `_newvec`만 가리키는 데이터만 지우는것이 아니라, `_vec`도 가리키고있는데이터를 함께 지우게됨
-> `_vec` dangling pointer! Queue소멸자에서 또 delete시, Double Free!