---
layout: post
published: false
title: Priorityqueue
date: 2026-09-28 23:10:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/Tree/HeapTree/HeaptreeCondition01.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>HeapTree Condition</strong> </td>  </tr> </table>

#### 새로운 값 추가
- 자식-부모 간의 구조 비교해서 Swap!

##### 최댓값 꺼내기
1. 가장 Root node를 꺼낸다
2. 가장 마지막 data -> Root로 옮긴다
3. 다시 Root와 자식 노드간의 비교 후 Swap!