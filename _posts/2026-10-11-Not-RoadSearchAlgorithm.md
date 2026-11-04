---
layout: post
published: false
title: 길찾기 알고리즘 (DFS, BFS, A*)
date: 2026-10-11 08:33:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
길찾기 알고리즘 대표적인 3가지를 알아보려 한다.
아래 인접행렬/인접 리스트를 이해해야지 다음 노드로 보내줄 수있는 길찾기 알고리즘을 이해할 수있다.

#### Keyword
- **DFS(Depth First Search)** :  
  깊이 우선 탐색 방식으로, 먼저 찾는것을 깊이우선으로 쭉 들어가는 방식이다.
- **BFS(Beardth First Search)** :
  너비 우선 탐색
- **Djikstra** :
  우선순위 큐를 활용한 길찾기
- **`A*`(Astar 알고리즘))** :
  Heuristic 함수를 사용하여 탐색 방향을 영리하게 결정한다.

### DFS :
DFS는 visited하지 않고, 보이면 바로 들어가는 길찾기이다.

``` cpp
#include <iostream>
#include <vector>
using namespace std;

struct Vertex {};

vector<Vertex> verties;
vector<vector<int>> adjacent(6);
vector<bool> visited;
void CreateGraph()
{
	verties.resize(6);

	adjacent[0] = { 1,3 };
	adjacent[1] = { 0,2,3 };
	adjacent[3] = { 4 };
	adjacent[5] = { 4 };
}

void DFS(int here)
{

	cout << "visited : " << here << endl;
	visited[here] = true;

	int _size = adjacent[here].size();
	for (int i = 0; i < _size; i++)
	{
		int there = adjacent[here][i];
		if (visited[there] == false)
			DFS(there);
	}
}

int main()
{
	CreateGraph();
	visited = vector<bool>(6, false);
	DFS(0);
}
```


### BFS :
BFS는 queue를 이용한 길찾기이다.


### Djickstra
Dijckstra는 Priorityqueue를 활용한 길찾기이다.

```cpp

vector<vector<int>> AdjacentMatrix(6, vector<int>(6, -1));
struct vertex {
	int data;
};
vector<vertex> verties;

struct VertexCost {
	VertexCost(int vertex, int cost) : _vertex(vertex), _cost(cost) {}
	//bool operator <(const VertexCost& other) const { return _cost < other._cost; }
	bool operator >(const VertexCost& other) const { return _cost > other._cost; }

	int _vertex;
	int _cost;
};

void CreateAdjacentMatrix()
{
	AdjacentMatrix = vector<vector<int>>(6, vector<int>(6, -1));
	AdjacentMatrix[0][1] = AdjacentMatrix[1][0] = 15;
	AdjacentMatrix[0][3] = AdjacentMatrix[3][0] = 35;
	AdjacentMatrix[1][2] = AdjacentMatrix[2][1] = 5;
	AdjacentMatrix[1][3] = AdjacentMatrix[3][1] = 10;
	AdjacentMatrix[3][4] = AdjacentMatrix[4][3] = 5;
	AdjacentMatrix[4][5] = AdjacentMatrix[5][4] = 5;
}

void Djickstra(int here)
{
	int size = AdjacentMatrix.size();
	
	priority_queue<VertexCost, vector<VertexCost>, greater<VertexCost>> pq;
	pq.push(VertexCost(here, 0));

	vector<int> Parent(6, -1);
	vector<int> Best(6, INT32_MAX);
	Parent[here] = here;
	Best[here] = 0;

	while (!pq.empty())
	{
		VertexCost VC = pq.top();
		pq.pop();
		int here = VC._vertex;
		int nowCost = VC._cost;
		if (Best[here] < nowCost ) continue; //이미 했던거는 스킵
		cout << "visit : " << here << endl;

		for (int there = 0; there < size; there++)
		{
			if (AdjacentMatrix[here][there] == -1) continue;
			int thereCost = AdjacentMatrix[here][there] + nowCost;
			if (thereCost >= Best[there]) continue;

			pq.push(VertexCost(there, thereCost));
			Parent[there] = here;
			Best[there] = thereCost;
		}
	}
}

int main()
{
	verties.resize(6);
	CreateAdjacentMatrix();
	Djickstra(0);

}

```