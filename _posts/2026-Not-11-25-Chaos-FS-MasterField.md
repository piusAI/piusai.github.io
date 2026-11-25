---
layout: post
published: false
title: FS_MasterField - Chaos Physics Field 구조
subtitle: Unreal engine RDG
date: 2026-11-25 01:01:00 +0900
description: RDG
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
`FS_MasterField`는 Unreal Engine에서 제공하는 사전 제작된 Chaos Physics Field Blueprint이다.


- Parent Class : `AFieldSystemActor`
- Component : `UFieldSystemComponent`
- 주요 기능 : External Strain, Internal Strain(Decay), Linear / Angular Velocity, Force, Noise

기본적으로 Geometry Collection에 **External Strain을 적용해 Cluster 연결을 끊고**, 분리된 파편에 Radial Angular Velocity와 Noise를 적용하여 파괴 효과를 연출한다.

Physics Field는 UFieldSystemComponent를 통해 Chaos Physics에 명령을 전달하며, 실제 물리 계산은 Chaos Solver가 수행한다.


### 1. Trigger 흐름

Hit Event를 이용하는 경우 다음과 같은 순서로 실행한다.

1. Hit Event
	- Object와의 충돌 감지
	- Hit Location 획득
	- 충격 위치에 `FS_MasterField Actor` 생성
2. Set Activation Type
	- Trigger로 설정
	- 자동 실행 아닌 명시적 활성화 방식
3. CE Trigger
	- `FS_MasterField`내부 Custom Event 호출
	- Field 활성화 및 파괴 처리 시작

※ Hit Event와 Spawn Actor는 구현 예시이며, 미리 배치한 Field Actor를 활성화할 수도 있다.

### 2. FS_MasterFiels 내부 처리
#### 2.1 Activation And Break

① Make Dynamic
- 필요한 경우 Kinematic 상태의 Rigid Body를 Dynamic 상태로 전환
- 파괴 처리와 별개로 물리 시뮬레이션 활성화를 제어

② External Strain
- Geometry Collection의 Cluster에 외부 Strain 적용
- Damage Threshold를 초과하면 연결이 끊어지며 파편 분리
- Magnitude, Radius, Falloff 등을 통해 영향 범위와 강도 조절

③ Internal Strain(Decay)
- Geometry Collection의 내부 Damage Threshold를 감소시킴
- 반복 적용시 점진적으로 약해지는 파괴 효과 구현 가능
- External Strain과 함께 사용 가능하며, 필수 단계는 아니다

#### 2.2 Make it Move
파괴된 파편의 움직임을 제어한다.

- Radial Velocity : Field 중심에서 바깥쪽으로 이동
- Directional Velocity : 지정된 방향으로 이동
- Angular Velocity : 파편 회전
- Force : 질량을 고려한 힘 적용
- Noise : 속도 및 힘에 무작위성 추가

기본 `FS_MasterField`는 Velocity 기반 움직임을 사용하며, 설정에 따라 Force 기반으로 변경 할 수 있다.

#### 3.3 Cleanup  & LifeSpan Logic
- Field 활성화 시간과 실행 횟수 제어
- 불필요한 Field Actor 정리
- Field Actor가 제거된다고 파괴ㅚㄴ 파편까지 자동으로 제거되는 것은 아니다.

`FS_MasterField`는 직접 파괴 물리 계산하는 Class는 아니고, Physics Field 명령 구성, Chaos Solver에 전달하는 BP Actor.


#### 전체 흐름 도식화

``` mermaid

flowchart TD
    A["Hit Event / Impact Location"]
    B["Spawn FS_MasterField"]
    C["Set Activation Type: Trigger"]
    D["CE Trigger"]

    subgraph FS["FS_MasterField Blueprint"]
        E["Activate And Break"]
        F["Make Dynamic (Optional)"]
        G["External Strain"]
        H["Internal Strain / Decay (Optional)"]
        I["Make it Move"]
        J["Linear / Angular Velocity<br/>Force / Noise"]
        E --> F --> G
        E --> H
        G --> I
        H --> I
        I --> J
    end

    K["UFieldSystemComponent<br/>Field Commands"]
    L["Chaos Physics Solver"]
    M["Cluster Breakage / Rigid Body Motion"]
    N["Field Actor Cleanup"]

    A --> B --> C --> D --> E
    J --> K --> L --> M
    J --> N

    style D fill:#17618D,color:#fff
    style K fill:#17618D,color:#fff
    style L fill:#375E46,color:#fff

```


---

Chaos Solver



``` mermaid

flowchart TD
    subgraph GT["① Game Thread"]
        A["Hit Event"]
        B["FS_MasterField<br/>CE Trigger"]
        C["UFieldSystemComponent"]
        D["FFieldSystemCommand 생성<br/>External Strain / Radial Velocity"]
        A --> B --> C --> D
    end

    subgraph BRIDGE["② Game ↔ Physics"]
        E["Command Buffer / Physics Proxy<br/>명령 및 물리 데이터 전달"]
    end

    subgraph PHYSICS["③ Chaos Physics Simulation"]
        F["FPBDRigidsSolver<br/>시뮬레이션 관리"]
        G["Field Command 평가<br/>Strain / Force / Velocity"]
        H["FPBDRigidsEvolution<br/>물리 Step 진행"]
        I["Collision Detection<br/>Broad / Narrow Phase"]
        J["Constraint Solving<br/>Contact / Joint / Cluster"]
        K["Position / Velocity 업데이트"]

        F --> G
        G --> H
        H --> I
        I --> J
        J --> K
    end

    L["④ Physics 결과 동기화<br/>GT / Component Transform"]
    M["Break Event / Niagara / Rendering"]

    D --> E --> F
    K --> L --> M

    style C fill:#1D5488,color:#fff
    style F fill:#14665C,color:#fff
    style H fill:#14665C,color:#fff
    style J fill:#79549D,color:#fff
```


