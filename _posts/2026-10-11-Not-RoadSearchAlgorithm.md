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

- **`A*`(Astar 알고리즘))** :
  Heuristic 함수를 사용하여 탐색 방향을 영리하게 결정한다.

### DFS 예시 코드:

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

`AFunc`의 정의 A.cpp에 있고, 호출은 B.Cpp가 한다.

### Compile 과정 전체 흐름

```
A.cpp ─전처리→ TU(A) ─컴파일→ A.obj ─┐
B.cpp ─전처리→ TU(B) ─컴파일→ B.obj ─┼─링킹→ 실행 파일
                    library ──────┘
```
- `ctrl + shift + b` (Build Solution)는 컴파일, 링킹 전체를 수행한다.
- 한파일만 컴파일 : `Ctrl + F7`
#### **01. Preprocessing, 전처리**
1. 주석 제거, 소스를 전처리 토큰으로 분해
2. 전처리 지시문 실행
  - `#include` : 헤더 내용을 텍스트 그대로 복사
  - `#define` : 매크로 치환
  - `#if / #ifdef` : 조건부 컴파일
  - `#pragma` 컴파일러별 명령처리(구현체마다 동작이 다름)
  - `#include` : 헤더 내용 복사(선언만!)
결과물이 **TU**이다.  
이 시점에서 `B.Cpp`는 `A.h`(헤더) 덕분에 `void AFunc();`같은 선언을 알게 된다.


#### **02. Compile, 컴파일**
1. 전처리 토큰들을 컴파일 토큰으로 변환
2. 구문 분석, 의미분석(문법과 타입 검사)
3. 필요한 Template Instance 생성(TU마다 수행)
4. 최적화 후 목적 코드 생성

- B.obj : `AFunc`호출 위치에 "정의는 외부에 있음"이라고 미해결 심볼로 기록
- A.obj : `AFunc`의 실제 구현 코드를 포함.

> 컴파일 단계의 문법오류는 컴파일 에러로 나오고 "정의 못 찾음"은 **Linker단계**에서 나옴!

#### **03. Linking, 링킹**
- 심볼연결해 실행 파일 생성
  B.obj : "`AFunc` 어디있어?" → A.obj : "여기있다" : 연결 완료
- 실행 파일이 되려면 `main`/`winmain` 이 반드시 필요
- 연결성에 따라 링커가 볼 수 있는 심볼이 달라짐
  - 외부 연결성(기본값) : 다른 TU에서도 참조 가능
  - 내부 연결성(`static`, 익명 namespace) : 해당 TU안에서만 보임, TU마다 별개의 복사본이 생긴다. [static 세가지 참고글](https://piusai.github.io/c++/2026/10/10/Static.html)
  
- 정의를 못 찾으면 링커 에러(MSVC : `LNK2019`, GCC : `undefined reference`)
- 같은 심볼의 정이가 둘 이상이라면 `LNK2005`(multiple definition)


#### 일반함수 vs inline 함수 / template

|          | 일반 함수             | inline 함수 / 템플릿                           |
| -------- | ----------------- | ----------------------------------------- |
| 헤더에 두는 것 | 선언                | 정의                                        |
| 정의 위치    | 한 cpp에만           | 헤더 (모든 TU가 봄)                             |
| 연결 방식    | 링커가 목적 파일 간 심볼 연결 | 각 TU에 정의가 있고, 중복은 허용(ODR 예외)되어 링커가 하나로 합침 |

일한 함수의 정의를 header에 두고, 여러 cpp에서 `#include`하면 정의가 중복되어 링커 에러가 난다.  
`inline`을 붙이는 이유가 이것.

> `inline`의 핵심은 **ODR 예외**이다.
> 실제 inline 확장 여부는 compiler가 최적화로 판단한다.
> 헤더에 `static void f(){}`를 두면 `LNK02005`는 나지 않지만, TU마다 복사본이 생긴다.