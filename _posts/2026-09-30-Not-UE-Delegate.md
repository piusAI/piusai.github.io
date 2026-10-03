---
layout: post
published: false
title: UE Delegate
subtitle: Constraint
date: 2026-09-21 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
Delegate는 다른 함수를 호출할 수 있도록하는 패턴이다.  
함수 객체와 비슷한 역할을 하고있다.  

Unreal engine에서는 매크로와 얽히고 섥혀있어 조금 헷갈리긴한다

복사해도 되나, 가급적 **참조** 전달로 하는것이 좋다.

### delegate
3박자가 준비되어야한다.
1. 부를 객체
2. 쓸 함수 - 매크로 인자갯수/return type 매칭!

| 단계      | 하는 일                           | 누가 하나                           |
| ------- | ------------------------------ | ------------------------------- |
| ① 타입 선언 | "이런 모양의 함수를 담는 그릇"을 정의         | 엔진 (`DECLARE_DELEGATE_...` 매크로) |
| ② 바인딩   | 그릇에 **내 함수 + 내 객체(this)** 를 담음 | `CreateRaw`, `BindRaw`          |
| ③ 실행    | 그릇 안의 함수를 실제로 호출               | 엔진 (`Execute()`)                |

우리 코드는 **②만**. ①은 엔진이 이미 해놨고, ③도 엔진이 알아서 한다.  
실행하는 쪽에서는 `CustomCBMenuExtender(...)`를 직접 호출하는 줄은 코드 어디에도 없다.  
바인딩 된 이후는 엔진이 대신 불러주기 때문입니다.


### 모르겠는것
`FExtender`, `FContentBrwoserMenuExtender_SelectedPaths`가 delegate 특정 변수