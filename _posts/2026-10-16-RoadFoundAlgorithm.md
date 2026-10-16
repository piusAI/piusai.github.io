---
layout: post
published: false
title: 길찾기 세가지 알고리즘 구현(DFS, BFS, A*)
date: 2026-10-09 21:13:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
DFS, BFS, A* 알고리즘에 대해 알아보자.

인접한 정보를 갖고 있어야하기 때문에 인접행렬에 대한 이해를 꼭 동반해야한다.
(포스팅 링크걸기)


### 1. DFS
- 깊이 우선 탐색 Depth-First Search
- 처음 찾는 것의 깊숙하게 들어가기
Tree 형태이니, 재귀함수로 들어감!!


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Algorithm/Graph/Graph02.png" alt="Graph01" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Graph01</strong> </td>  </tr> </table>

위 그래프를 cpp로 구현 해보자

#### 1-1. 인접 리스트를 활용한 DFS 

``` cpp
#include <iostream>
#include <stack>
using namespace std;
#include <vector>

struct Vertex{ int _data; };

vector<Vertex*> verties;
vector<bool> visited(4, false);

vector<vector<int>> adjacentList(4); 
void CreateAdjacentList() {
	adjacentList[0] = { 1, 3 };
	adjacentList[1] = { 0, 2, 3 };
	adjacentList[3] = { 0 };
	adjacentList[4] = { 2 };

}

void DFSList(int here)
{
	visited[here] = true;
	cout << " visit ! : " << here << endl;
	int _size = adjacentList[here].size();
	for (int v = 0; v < _size; v++)
	{
		int there = adjacentList[here][v];
		if (!visited[there]) DFSList(there);
	}
}


int main()
{
	verties.resize(4);
	CreateAdjacentList();
	DFSList(0);

}
```


#### 1-2. 인접 행렬을 활용한 DFS 

``` cpp
#include <iostream>
#include <stack>
using namespace std;
#include <vector>

struct Vertex{ int _data; };

vector<Vertex*> verties;
vector<bool> visited(4, false);
vector<vector<int>> AdjacentMatrix(4, vector<int>(0));

void CreateAdjacentMatrix(){

	AdjacentMatrix = {
		{0, 1, 0, 1, 0},
		{1, 0, 1, 1, 0},
		{0, 0, 0, 0, 0},
		{1, 0, 0, 0, 0},
		{0, 0, 1, 0, 0}
	};
}

void DFSMatrix(int here)
{
	visited[here] = true;
	cout << "visited : " << here<<endl;

	int _size = AdjacentMatrix.size();
	for (int v = 0; v < _size; v++)
	{
		if (AdjacentMatrix[here][v] == 1 && !visited[v])
			DFSMatrix(v);
	}
}

int main()
{
	verties.resize(4);
	CreateAdjacentMatrix();
	DFSMatrix(0);
}
```

인접 리스트 방식과 다를것 없고, 방문하는 방법만 다르다


### 인접 리스트 vs 인접 행렬 in DFS

- 인접 행렬 시간 복잡도 : O( $V^2$) 
vertex를 v번 반복해 방문하고(`for`),  v번 더  `if(Adjacent[here][v]==1)`반복

- 인접 리스트 시간 복잡도 : O(`V+E`)
vertex를 v번 반복(`for`)하고, E(`if(Adjacent[here][v])`)


---
