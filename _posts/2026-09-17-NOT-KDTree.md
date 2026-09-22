---
layout: post
published: false
title: KDTree
date: 2026-09-16 19:10:00 +0900
description: 스택 메모리
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
k차원 공간의 점들을 효율적으로 탐색하기 위한 Binary Tree 자료구조.



### 핵심 IDEA
1. 축 번갈아 분할 : Tree depth에 따라 분할 기준 축(axis)를 번갈아 가며 사용
2. 중앙값으로 분할 : 해당 축을 기준으로 점들을 정렬한뒤, 중앙값(Median)을 루트로 삼고, 그보다 작은 점들은 왼쪽 subtree, 큰 점들은 오른쪽 Subtree로 재귀적 구성
3. 탐색시 가지치기 : 최근접 이웃 탐색시, 분할축과 Querry점 사이의 거리가 현재까지 찾은 거리보다 크면 반대편 서브트리는 아예 탐색 하지 않아도 됨!
   → KD-Tree가 빠른 이유!'


### 시간 복잡도
- O(nLogn)
- 최근접 이웃 탐색 : 평균 O(log n), 최악 : O(n)


``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <memory>
#include <limits>
#include <cmath>
#include <random>

struct Point {
	double x,y ;
};

//uclid distance 제곱 (sqrt 생략으로 성능최적화)
double distSq(const Point& a, const Point& b)
{
	double dx = a.x - b.x;
	double dy = a.y - b.y;

	return dx * dx + dy * dy;
}

// ============ KD - tree Node ============
struct KDNode {
	KDNode(const Point& p) : point(p), left(nullptr), right(nullptr) {}

	Point point;
	std::unique_ptr<KDNode> left;
	std::unique_ptr<KDNode> right;
};

class KDTree {
public:
	KDTree() = default;

	//점들 집합 tree 구성
	void build(std::vector<Point> points)
	{
		root = buildRecursive(points, 0, (int)points.size(), 0);
	}
	Point nearest(const Point& query) const {
		if (!root) throw std::runtime_error("Tree 빔?!");
		const KDNode* best = nullptr;
		double bestDistSq = std::numeric_limits<double>::max();
		nearestRecursive(root.get(), query, 0, best, bestDistSq);
		return best->point;
	}

	//트리 들여쓰기 형태로 출력
	void print() const {
		printRecursive(root.get(), 0, 0);
	}
private:
	//최근접 이웃 탐색 재귀함수
	//best l-value로 포인터 원본 갈아끼운다?
	void nearestRecursive(const KDNode* node, const Point& query, int depth,
		const KDNode*& best, double& bestDistSq) const {
		if (!node) return;

		double d = distSq(node->point, query);
		if (d < bestDistSq)
		{
			bestDistSq = d;
			best = node;
		}
		int axis = depth % 2;
		double diff = axisValue(query, axis) - axisValue(node->point, axis);

		//쿼리있는 쪽(가까운) subtree 먼저 탐색
		const KDNode* nearChild = diff < 0 ? node->left.get() : node->right.get();
		const KDNode* farChild = diff < 0 ? node->right.get() : node->left.get();

		nearestRecursive(nearChild, query, depth + 1, best, bestDistSq);

		//가지치기 : diff^2가 현재 최선 거리보다 작을떄만
		if (diff * diff < bestDistSq)
		{
			nearestRecursive(farChild, query, depth + 1, best, bestDistSq);
		}

	}
	void printRecursive(const KDNode* node, int depth, int axis) const {
		if (!node) return;
		std::cout << std::string(depth * 2, ' ')
			<< "(" << node->point.x << ", " << node->point.y << ")"
			<< "[axis=" << (axis == 0 ? "x" : "y") << "]\n";
		printRecursive(node->left.get(), depth + 1, 1 - axis);
		printRecursive(node->right.get(), depth + 1, 1 - axis);


	}
	static bool compareByAxis(const Point& a, const Point& b, int axis) {
		return axis == 0 ? (a.x < b.x) : (a.y < b.y);
	}
	static double axisValue(const Point& p, int axis)
	{
		return axis == 0 ? p.x : p.y;
	}
	//[lo, hi) 구간 points 재귀적으로 분할후 SubTree 구성!
	std::unique_ptr<KDNode> buildRecursive(std::vector<Point>& points, int lo, int hi, int depth)
	{
		if (lo >= hi) return nullptr;
		int axis = depth % 2; //2차원 0 :x, 1 : y
		int mid = lo + (hi - lo) / 2;

		// nth_element로 중앙값 O(n) 평균에 찾음 - 전체 정렬보다 빠르다
		std::nth_element(points.begin() + lo, points.begin() + mid, points.begin() + hi,
			[axis](const Point& a, const Point& b) {return compareByAxis(a, b, axis); });
		auto node = std::make_unique<KDNode>(points[mid]);
		node->left = buildRecursive(points, lo, mid, depth + 1);
		node->right = buildRecursive(points, mid + 1, hi, depth + 1);
		return node;
	}

	

private:
	std::unique_ptr<KDNode> root;
};

Point bruteForceNearest(const std::vector<Point>& points, const Point& query)
{
	Point best = points[0];
	double bestD = distSq(points[0], query);

	for (const auto& p : points) {
		double d = distSq(p, query);
		if (d < bestD) { bestD = d; best = p; }
	}
	return best;

}

int main()
{
	std::vector<Point> points = {
	 {2, 3}, {5, 4}, {9, 6}, {4, 7}, {8, 1}, {7, 2}
	};

	KDTree tree;
	tree.build(points);

	std::cout << "===== KD-tree 구조 =====\n";
	tree.print();

	Point query{ 9, 2 };
	Point result = tree.nearest(query);
	std::cout << "\n쿼리 점: (" << query.x << ", " << query.y << ")\n";
	std::cout << "최근접 이웃: (" << result.x << ", " << result.y << ")\n";

	// ---------- 랜덤 데이터로 브루트포스와 결과 비교 ----------
	std::cout << "\n===== 랜덤 데이터 검증 (1000개 점, 100번 쿼리) =====\n";
	std::mt19937 rng(42);
	std::uniform_real_distribution<double> dist(0.0, 1000.0);

	std::vector<Point> randomPoints;
	for (int i = 0; i < 1000; ++i)
		randomPoints.push_back({ dist(rng), dist(rng) });

	KDTree bigTree;
	bigTree.build(randomPoints);

	bool allMatch = true;
	for (int i = 0; i < 100; ++i) {
		Point q{ dist(rng), dist(rng) };
		Point kdResult = bigTree.nearest(q);
		Point bfResult = bruteForceNearest(randomPoints, q);
		if (distSq(kdResult, q) != distSq(bfResult, q)) {
			allMatch = false;
			std::cout << "불일치 발견! query=(" << q.x << "," << q.y << ")\n";
		}
	}
	std::cout << (allMatch ? "모든 결과가 브루트포스와 일치합니다. ✅\n"
		: "일부 결과가 불일치합니다. ❌\n");

	return 0;
}

```
Claude와 함께한 KD-Tree cpp 구현

추후 보완하기
- `void nearestRecursive(.., const KDNode*& best..)`
  L-value, 재귀 호출을 여러번 겹쳐 하더라도 계속 같은 변수 취급 해서 업데이트 하도록하기위해!
- [axis] capture를 해야하는이유