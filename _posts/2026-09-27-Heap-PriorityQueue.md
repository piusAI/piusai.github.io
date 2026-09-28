---
layout: post
published: true
title: "Heap과 Priorityqueue : 자료구조에서 어디에 위치할까?"
date: 2026-09-28 23:10:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
자료구조가 약한 터라 정리가 필요한 시점이다.  
BST, QuadTree와 같이 Terrain Tessellation 최적화 알고리즘에서 자주 쓰이는 Tree 계열을 이해하려면 먼저 **거시적인**지도가 필요하다.

### 1. 거시적으로 보는 자료구조
<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/Tree.png" alt="Tree" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree Condition</strong> </td>  </tr> </table>
- **Tree(일반)**: 자식 개수에 제한이 없다.
- **Binary Tree(이진 트리)**: 자식이 최대 2개다.
- **완전 이진 트리**: 이진 트리 중에서 위→아래, 왼쪽→오른쪽으로 **빈틈없이** 채운 모양이다.
- **힙(Heap)**: 완전 이진 트리에 "부모 ≥ 자식"(최대 힙) 또는 "부모 ≤ 자식"(최소 힙) 규칙을 얹은 것이다.
- **BST**: 이진 트리에 "좌 < 부모 < 우" 규칙을 얹은 것이다. 힙과는 별개의 갈래다.


#### PriorityQueue는 어디에 있나?

그림에는 PriorityQueue가 힙 안에 적혀 있지만, 엄밀히는 **층위가 다르다**.

|층위|이름|정체|
|---|---|---|
|추상 자료형(ADT)|PriorityQueue|"우선순위 높은 것부터 꺼낸다"는 **약속(인터페이스)**|
|구체 자료구조|Heap|그 약속을 **구현하는 방법** (가장 흔함)|
정렬된 배열이나 BST로도 우선순위 큐를 만들 수 있다. 다만 힙이 삽입·삭제 모두 O(log n)이고 최댓값 확인이 O(1)이라 표준처럼 쓰인다.  
Stack, Queue가 ADT이고 배열/리스트가 구현인 것과 같은 관계다.

### 2. 일반적인 Tree와 노드
노드는 데이터와 자식으로 가는 연결을 묶은 단위다.

```cpp
//일반 트리 : 자식 수 제한 x
struct Node{
	int data;
	vector<Node*> children;
}

//이진 트리 : 자식 최대 2개
struct BNode{
	int data;
	BNode* left;
	BNode* right;
	}
```
Quad Tree는 `children` 이 **최대 4개**인 트리이다.  
지형을 4분할하며 내려가는 구조라서 **Terrain 최적화**에 잘 맞겠다.

### 3. 핵심 아이디어 : 포인터 없이 배열 인덱스만으로 트리를 표현해보자!
완전 이진트리는 빈틈없이 채워지므로 위->아래, 왼쪽->오른쪽 순서대로 배열(Vector)에 그대로 넣을 수있겠다.  
그러면 노드사이의 연결(Pointer)가 필요 없고, **Index 계산만**으로 부/자 관계를 알 수 있겠다.

```
트리                       배열 (인덱스)
        1                  [ 1 | 3 | 2 | 7 | 4 ]
       / \                   0   1   2   3   4
      3   2
     / \
    7   4
```

#### 배열에서의 Index 공식

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/HeapTreeArray.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree in Array</strong> </td>  </tr> </table>
- i번 노드 자식 왼쪽 : $(2*i) +1$
- i번 노드 자식 오른쪽 : $(2*i)+2$
- i번째 부모 : $floor(i-1/2)$

CBT Algorithm2에서와 같이 인덱싱으로 표현 가능!

일반 이진트리는 중간이 빌 수있기때문에 **완전 이진트리**라서 이 Index 공식이 성립한다!  
- Heap이 배열로 구현되는 이유!

### 4. Heap의 두가지 조건

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/HeaptreeCondition01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree Condition</strong> </td>  </tr> </table>
**1법칙**
- 자식 노드 < 부모노드 (maxheap 기준)
- 마지막 깊이 빼고는 모두 차있다 (완전 BT)
- 마지막 레벨에서의 노드는 왼쪽부터 채우기
**2법칙**
- 노드 갯수알면 트리 구조 확정 가능
- $2^h-1$가 아니더라도 무조건 트리 구조 확정 가능!

BST는 자식 왼쪽 Subtree< 부모 < 오른쪽 subtree

- 부모/자식 관에만 대소관계가 있고, **형제**끼리는 **노상관**!  
그래서 힙은 BST 처럼 ***정렬된 상태***가 아니다!  
**루트 하나(최대/최소)만**이 보장된다.

Priority Queue는 아래 글에서 조금 더 자세히 다루겠다.  
(글 작성 예정)

### 5. Heap vs Stack

|특징|Heap|Stack|
|---|---|---|
|분류|우선순위 큐를 구현하는 자료구조|LIFO ADT|
|꺼내는 기준|우선순위가 가장 높은 것|가장 나중에 넣은 것|
|내부 구조|완전 이진 트리 (배열로 저장)|배열 또는 연결 리스트|
|접근|루트만 직접 접근|top만 접근 가능|
|삽입/삭제|O(log n)|O(1)|
|용도|우선순위 큐, 힙 정렬, 다익스트라|DFS, 함수 호출 관리, 되돌리기|

> **주의: 이름이 같은 다른 개념**  
> 메모리의 **힙 영역**(동적 할당, `new`)과 **스택 영역**(지역 변수, 함수 호출)은 이 글의 자료구조와 **이름만 같다.** 메모리 구조 이야기와 섞이지 않게 구분하기!!

### 6. 정리

- Tree ⊃ Binary Tree ⊃ 완전 이진 트리 ⊃ Heap
- PriorityQueue는 ADT, Heap은 그 구현체
- 완전 이진 트리라서 **포인터 없이 배열과 인덱스 계산**으로 표현할 수 있다.
- 힙은 정렬된 구조가 아니라 **루트 하나만 보장**하는 구조다.

