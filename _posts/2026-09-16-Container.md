---
layout: post
published: true
title: 기본 자료구조 (Vector, Queue 구현)
date: 2026-09-16 19:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
동적 배열을 만들기 위해서 필요한것들

push_back, begin, opertor[], 추가했다.


#### Vector 구현 cpp

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
		if (&other == this) return *this;

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

#### Queue 구현
#####  01 `Vector<T>*`를 활용한 Queue

``` cpp
template <typename T>
class Queue{

public:
	explicit Queue(int capacity) : _data(new Vector<T>(capacity)), _capacity(capacity)
	{
		_data->resize(capacity); //원형 큐 다 채워주기!

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
