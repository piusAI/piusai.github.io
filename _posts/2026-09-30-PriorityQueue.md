---
layout: post
published: true
title: Priorityqueue의 이해와 구현
date: 2026-09-30 21:13:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
PriorityQueue, 우선순위 큐는 HeapTree의 구현이다.

구현에 앞서 PriorityQueue의 컨셉을 먼저 이해해보자.

자료구조에서 포인터 없이 우선순위 큐를 Vector(배열)로 표현할 수있다고 했었는데,
이를 Index로 옮길 수있다.

### 1. Heap의 두가지 Main 조건
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/HeaptreeCondition01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree Condition</strong> </td>  </tr> </table>
**1법칙**
- 자식 노드 < 부모노드 (Maxheap 기준)
- 마지막 깊이 빼고는 모두 차있다 (완전 BT)
- 마지막 레벨에서의 노드는 왼쪽부터 채우기
**2법칙**
- 노드 갯수알면 트리 구조 확정 가능
- $2^h-1$가 아니더라도 무조건 트리 구조 확정 가능!

- 부모/자식 에만 *대소관계*가 있고, **형제**끼리의 *대소관계*는 **노상관**! 
그래서 힙은 BST 처럼 형제간***정렬된 상태***는 아님!

### 2. 배열에서의 Index 공식
아래의 `A[0], A[1]`과 같이 Tree를 Vector로 표현하면 인덱스 Number가 존재한다.  
이것을 자식-부모 인덱스 공식을 만들 수있다.
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/HeapTreeArray.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree in Vector[array]</strong> </td>  </tr> </table>
- i번 노드 자식 왼쪽 : $(2*i) +1$
- i번 노드 자식 오른쪽 : $(2*i)+2$
- i번째 부모 : $floor(i-1/2)$

[CBT Algorithm](https://piusai.github.io/engine/2026/09/29/Concurrent-binaryTrees) 두번째 알고리즘에서와 같이 인덱싱으로 표현 가능!

일반 이진트리는 중간이 빌 수있기때문에 **완전 이진트리**라서 이 Index 공식이 성립한다!  
- Heap이 배열로 구현되는 이유!

### 3. 새로운 값 추가
- 자식-부모 간의 구조 비교해서 Swap!
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/Add.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>PQ 노드 추가</strong> </td>  </tr> </table>
1. 트리 구조에 맞게 정렬 `push_back`으로 가장 마지막 인덱스에 들어간다
2. 부모 노드와 비교해서 대소관계 Swap을 한다(Loop)
3. Loop 조건
	3.1 자식이 작을때만 Swap
	3.2 Index가 root(0)에 도달시

### 4. 최댓값 꺼내기
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/MaxDel.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>PQ 최댓값 제거</strong> </td>  </tr> </table>
1. 가장 Root node를 꺼낸다 - `A[0]` `pop_back`으로 제거한다
2. 가장 마지막 data -> Root로 옮긴다
3. 옮긴 Root와 자식 노드간의 비교 후 Swap!
	3.1 자식이 작을때만 Swap
	3.2 Left자식, Right 자식중 더 큰쪽과 Swap
	3.3 계속 Loop 반복
4. Loop 조건
	4.1 부모가 크다면 비교 중지
	4.2 또는 Leaf 노드라면 확인 중지


### 5. CPP 코드 구현

``` cpp


template <typename T>
class PriorityQueue{
public:
	PriorityQueue() {}
	~PriorityQueue(){
		for (int i = 0; i < _vec.size(); i++)
			delete _vec[i];
	}
	PriorityQueue(const PriorityQueue&) = delete;
	PriorityQueue& operator=(const PriorityQueue&) = delete;
	void Push(const T& data){

		_vec.push_back(new T(data));
		int current = (int)_vec.size() - 1; //방금 넣은 마지막 원소

		while(current>0)
		{
		int parent = int(current -1) / 2;
		if ( *_vec[parent] > *_vec[current]) break;

		::swap(_vec[current], _vec[parent]);
		current = parent;
		}
	}
	const T& Top()const
	{
		assert(!_vec.empty());
		return *_vec[0];
	}
	void Pop()
	{
		if (_vec.empty()) return;
		::swap(_vec[0], _vec.back()); //루트 맨 뒤로
		delete _vec.back(); //누수 방지
		_vec.pop_back();

		int size = (int)_vec.size();
		int current = 0; //top

		while(true)
		{
		int Leftchild = current * 2 + 1;
		int Rightchild = current * 2 + 2;
		int largest = current;
	
		if (Leftchild < size && *_vec[largest] < *_vec[Leftchild])
			largest = Leftchild;
		if (Rightchild < size && *_vec[largest] < *_vec[Rightchild])
			largest = Rightchild;
		if (largest == current) break; //부모가 가장크면 빠져나오라!
	
		::swap(_vec[current], _vec[largest]);
		current = largest;
		}
	}
	

	void Print() const
	{
		for(int i = 0 ; i < _vec.size(); i ++)
		cout << *(_vec[i]) << endl;
	}
public:
	vector<T*> _vec;

};
int main()
{

	PriorityQueue<int> pq;
	pq.Push(10);
	pq.Push(40);
	pq.Push(50);
	pq.Push(60);
	pq.Push(70);

	
	int TQ = pq.Top();
	cout <<"TOP :" << TQ << endl;
	pq.Pop();
	
	pq.Print();

}

```