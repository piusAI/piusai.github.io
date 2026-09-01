---
layout: post
published: true
title: FRDGBuilder를 알고싶다
subtitle: Constraint
date: 2026-08-26 19:37:00 +0900
description: FRDGBuilder
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
FRDGBuilder는 RDG에서의 RenderPipeline 제어 객체이다.

### **FRDGBuilder** (Render Dependency GraphBuilder)
- 정의 : UE 차세대 Render Pipeline 제어 객체,
- 핵심 기능 : Render Pass, Resource(Texture, Buffer)간의 의존성 관계 Graph로 구성한뒤, 
  최적화(Cull, Compile)를 거쳐 일괄 실행(Execute)하는 Scheduler 및 **오케스트레이터 Class!**

- DX PSO / Context와 유사:
  Dx에서의 `PSO(Pipeline State Object)`나 Context에 `BlendState`, `DepthStencilState`, `RasterizerState`를 미리 세팅해놓고 `DrawCall`을 날리는 개념과 구조적으로 유사!
- 특징:
  -  Resource Barrier와 수명은 각 `AddPass()` Call에 제공되는 parameter struct(`FParameters`)의 RDG parameter에서 자동으로 파생됨.  
  - Builder 객체는 Stack에서 생성되어 Scope를 벗어나 소멸키전에 반드시 `Execute()`가 호출 되어야함

역시 한번더 RDG의 Compute shader 단계를 설명 하지 않을 수 없겠다.  
FRDGBuilder에 초점을 두어 lifeCycle을 알아보자.
### **RDG** 3단계 LifeCycle (Pipeline)

```
[01. GraphBuild 단계] : AddPass()로 노드/dependency 등록
[02. Compile & Allocate Stage] : Cull, LifeTime, Barrier 계산 및 할당
[03. Execute Stage] : 실제 RHI Command Dispatch 
```

#### 01 Graph Build 단계
- `GraphBuilder.AddPass(..)`를 호출 할때마다 실행이 되는것이 아니라,  
  `FRDGPass`노드를 Graph에 등록만 한다.  
-  Pass에 전달되는 Parameter 구조체(`FParameters`)를 분석해서,  
   어떤 리소스를 읽고 쓰는지 자동으로 파악,  
-   이를 통해 Pass-Resource간 의존성 간선(Dependency Edge)이 Graph에 순차적으로 축적.
#### 02 Execute() 내부 Pipeline
`GraphBuilder.Execute()`가 호출되는 시점, 수집된 Graph를 기반으로 아래 과정 차례로 진행

 1. **Compile** 단계 -그래프 순회 및 최적화
    - **Graph Traversal** :  그래프를 Travarsal(순회)하면서 pass/Resource culling 여부 결정
    - **Transient Lifetime 계산** :각 Resource의 LifeTime 계산 언제 풀에서 할당 / return 결정
    -  **Barrier 최적화** : Resource 전환(Barrier) 시점 계산
2. **Collect/Allocate** 단계 - 리소스 할당
	- Culling에서 살아남은 Resource에 한해 Pooled RHI 리소스 지연 할당(Lazy Allocation)
	- 계산된 정보를 바탕으로 일괄 적용할 Resource Transition command 준비
3. **Execute** 단계 - 실제 RHI 실행
    - Compiled 된 순서대로 Pass 순회 하면서 각 패스 에 등록된 람다 실제 호출
      `[](FRHICommandListImmediate& RHICmdList){...}`

``` cpp
[](FRHIComputeCommandList& RHICmdList){ ... }
```
    - 필요한 시점에 계산해둔 resource barrier 삽입
    - pass가 병렬 실행 플래그를 갖고있으면 Task Grpah 이용해 로 Command List 생성을
      여러 Threads로 분산하여 병렬 처리

---
### FParameters
Shader 클래스 안에서 매크로로 선언해둔 **GPU 파라미터 구조체**의 타입 이름


