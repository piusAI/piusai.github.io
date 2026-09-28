---
layout: post
published: false
title: SnowDeformation Plugin 아키텍쳐 정리
subtitle: Constraint
date: 2026-09-21 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---
# Snow Deformation 시스템 아키텍처 정리

> `DeformableSnowSystem`
> 플러그인의 전체 구조를 이해하고, 다른 프로젝트에 직접 이식할 수 있도록 정리
> Codex의 에디터 확인 결과 표기:
> ✅ 에디터에서 직접 확인됨  
> 🔶 이름·구조로 추정  
> ❓ 아직 미확인

---

## 1. 한 줄 요약

**Trace Component가 "여기에 자국 찍어줘"라고 요청 → `BP_SnowGenerator`가 그 모양을 캡처해 Render Target에 누적 → 눈 머티리얼이 그 RT를 읽어 표면을 변형한다.**

핵심 원칙 두 가지:
1. 컴포넌트는 눈 머티리얼을 **직접 건드리지 않는다.** 요청(데이터)만 Generator에 넘긴다.
2. 눈 머티리얼은 게임 로직을 모른다. **결과 텍스처(`RT_Snow`)만 읽는다.**

Fab 설명 기준으로 이 시스템이 지원하는 효과는 다음과 같다.

| 효과                | 관련 요소 (추정 포함)                          |
| ----------------- | -------------------------------------- |
| 깊이감 있는 트레일        | `RT_Snow`, `SnowDeformationOffset`     |
| 자국 가장자리 융기        | `SnowRidgeMask`                        |
| 부드러운 가루눈 Fall-off | Hardness, `MergeRT`의 합성                |
| 시간에 따른 서서히 복원     | `SetSnowAttenuation`, `RT_SnowHistory` |

---

## 2. 전체 구조도

```
[ 자국을 남기는 Actor (캐릭터 등) ]
   ├─ BP_FootTraceComponent      (SceneComponent)  발 위치 기반
   └─ BP_CustomTraceComponent    (SceneComponent)  일반 Actor 기반
            │
            │  조건(이동량/속도) 통과 시 FCustomTrace 생성
            ▼
     ┌──────────────────────────────────────────┐
     │  BP_SnowGenerator  (Actor, 맵에 1개 배치)   │
     │   - 요청 수집 (addFootPrint)                │
     │   - SceneCaptureComponent2D                │
     │   - SceneCaptureForEachActor                │
     │   - MergeRT / CopyToHistory                │
     │   - SetSnowAttenuation                     │
     │   - MPC_Snow 값 갱신                        │
     └───────┬──────────────────────────────────┘
             │
   ┌─────────┴─────────── Render Target 4개 ──────────┐
   │  RT_Capture (256²)  → 이번 프레임 자국 1개         │
   │  RT_Intermediate (2048²) → 합성 중간 작업장 🔶     │
   │  RT_Snow (2048²) → 현재 눈 상태 (최종 결과)         │
   │  RT_SnowHistory (2048²) → 직전 상태                │
   └─────────┬───────────────────────────────────────┘
             │ RT_Snow만 머티리얼로 전달
             ▼
     M_Snow (Master Material)
       └─ MF_DeformationProcess (Material Function)
            ├─ SnowDeformationOffset  → World Position Offset
            ├─ SnowDeformationNormal  → Normal
            ├─ SnowTrailMask
            ├─ SnowTrackedMask
            ├─ SnowRidgeMask
            └─ NaniteDisplacement
```

프로젝트 자산 이름과 종류는 다음과 같다.

| 이름                                                           | 종류                            | Member Reference                                                        |
| ------------------------------------------------------------ | ----------------------------- | ----------------------------------------------------------------------- |
| `BP_SnowGenerator`                                           | Actor (Blueprint)             | MPC_Snow, M_Merge, RT_Capture, RT_Intermediate, RT_Snow, RT_SnowHistory |
| `BP_CustomTraceComponent`, `BP_FootTraceComponent`           | SceneComponent (Blueprint)    | SnowGenerator                                                           |
| `FCustomTrace`, `FFootPrintInfo`                             | Struct                        | RT                                                                      |
| `MPC_Snow`                                                   | Material Parameter Collection |                                                                         |
| `M_Snow` (`M_Snow_Inst`)                                     | Material                      | RT_Snow, MF_DeformationProcess                                          |
| `MF_DeformationProcess`                                      | Material Function             |                                                                         |
| `M_Merge`                                                    | 합성용 Dynamic Material Instance | RT_SnowHistory                                                          |
| `RT_Capture`, `RT_Intermediate`, `RT_Snow`, `RT_SnowHistory` | Render Target                 |                                                                         |

---

## 3. 구성요소별 상세
### 3.1 Trace Component (입력 측)
#### BP_CustomTraceComponent ✅

- 캡처하려는 Actor에 **부착**해서 쓴다. (Blueprint 내부 설명에도 명시)
- 변수: `CaptureRT`, `Hardness`, `Compensation`, `OldPos`, `CurPos`
- `BeginPlay`에서 `BP_SnowGenerator`를 찾아 `Generator` 변수에 저장
- `AddCustomTrace` 함수에서 `FCustomTrace` 구조체를 만들어 Generator에 등록
    

#### BP_FootTraceComponent 🔶

- 발 위치 추적용 별도 컴포넌트. Generator 쪽 `addFootPrint`로 보낸다.
- 내부 그래프의 모든 핀 연결은 판독하지 못했다. CustomTrace와 **같은 방식이라고 단정하지 않는다.**

|        | CustomTrace                   | FootTrace                     |
| ------ | ----------------------------- | ----------------------------- |
| 용도     | 일반 Actor의 캡처 정보로 자국 생성        | 발의 이동·위치로 발자국 생성              |
| 최종 목적지 | `SnowGenerator::addFootPrint` | `SnowGenerator::addFootPrint` |
| 내부 동일성 | ❓ 확인 필요                       | ❓ 확인 필요                       |

> 사용자 메모: "Actor Class를 get해서 line trace를 돌려 add custom trace로 넘긴다"는 흐름은 두 컴포넌트가 비슷하게 갖는다고 했다. 이 부분(어느 쪽이 trace를 돌리고 어느 쪽이 Actor를 넘기는지)은 Codex 그래프 판독에서 끝까지 확인되지 않았다.

### 3.2 FCustomTrace 구조체 ✅
"자국 요청" 데이터 구조체

| 필드               | 용도                    |
| ---------------- | --------------------- |
| Capture Location | 자국을 찍을 월드 위치          |
| Capture RT       | 캡처에 사용할 Render Target |
| Actor to Draw    | 캡처 대상 Actor           |
| Rotation         | 자국 방향                 |
| RT Size          | 캡처 크기                 |
| Hardness         | 자국의 강도/선명도            |
| Compensation     | 보정값                   |

`FFootPrintInfo`는 발자국용 구조체로, 이름상 `FCustomTrace`와 짝을 이루는 두 번째 구조체다. 필드 목록은 이번 확인에서 나오지 않았다. ❓

### 3.3 BP_SnowGenerator (중앙 처리자) ✅

- `SceneCaptureComponent2D`를 가진다. 맵에 **반드시 배치**해야 한다.
- `BeginPlay`의 Sequence에서 초기화·갱신 작업을 분기한다.
- 기본 속성: `RT Snow`, `RT Snow History`, `RT Intermediate`, `RTResolution` (기본 2048×2048)
- 확인된 함수:

| 함수                         | 역할                                                     |
| -------------------------- | ------------------------------------------------------ |
| `SceneCaptureForEachActor` | 요청받은 Actor마다 씬 캡처를 수행 (`RT_Capture`에 기록) 🔶            |
| `MergeRT`                  | `RT_Snow`를 Clear한 뒤 다시 그려 새 상태 생성 ✅(Clear) 🔶(이후 Draw) |
| `CopyToHistory`            | `RT_Snow`를 Draw Texture 입력으로 읽어 히스토리로 복사 ✅(입력) 🔶(대상)  |
| `SetSnowAttenuation`       | `DeltaTime × Trail Attenuation`을 Clamp하는 시간 감쇠 계산 ✅    |

`MergeRT` 안에서 `M_Merge` 오브젝트에 **Vector Parameter를 세팅**하는 연결이 확인됐다. 즉 합성은 머티리얼(`M_Merge`)로 그리는 방식이다.

---

## 4. Render Target 4개

| RT                | 크기 / 포맷            | 역할                             | 확실성          |
| ----------------- | ------------------ | ------------------------------ | ------------ |
| `RT_Capture`      | 256×256, R16f      | 자국 **하나**를 캡처한 작은 입력 텍스처 (도장)  | ✅ 설정 / 🔶 역할 |
| `RT_Intermediate` | 2048×2048, RGBA16f | 합성·가공의 중간 작업장                  | ✅ 설정 / 🔶 역할 |
| `RT_Snow`         | 2048×2048          | **현재** 눈 변형 상태. 머티리얼이 읽는 최종 결과 | ✅ 속성 / 🔶 역할 |
| `RT_SnowHistory`  | 2048×2048          | **직전** 상태. 다음 프레임 합성의 기준       | ✅ 속성 / 🔶 역할 |
**비유**:
- `RT_Capture` = 이번에 찍은 도장 하나
- `RT_Intermediate` = 도장을 종이에 찍기 전 작업대
- `RT_Snow` = 지금 종이(눈)에 찍혀 있는 상태
- `RT_SnowHistory` = 방금 전 종이의 사본

### 4.1 프레임 단위 데이터 흐름 (추정 포함)

```
① 요청 도착      FCustomTrace 목록이 Generator에 쌓임
② 캡처          SceneCaptureForEachActor → RT_Capture (자국 모양)
③ 합성 준비      RT_Snow를 Clear (MergeRT)
④ 합성          M_Merge로 [RT_Capture + RT_SnowHistory(+감쇠)] 를 그림
                  → RT_Intermediate 를 거칠 수 있음 🔶
⑤ 결과 확정      RT_Snow 갱신
⑥ 기록          CopyToHistory: RT_Snow → RT_SnowHistory (다음 프레임용)
⑦ 표시          M_Snow가 RT_Snow를 읽어 변형
```

- "시간이 지나면 눈이 복원된다"는 효과는 ④에서 History 값에 **감쇠**(`SetSnowAttenuation`)를 곱해 조금씩 약하게 만드는 방식으로 구현됐을 가능성이 높다. 🔶
- ③에서 `RT_Snow`를 지우고 ④에서 History + 새 자국으로 다시 채우는 **"지우고 다시 그리기"** 패턴이다. Clear 자체는 합성이 아니므로, 실제 동작은 **그 뒤에 어떤 텍스처를 그리느냐**가 결정한다. ❓
    

---

## 5. MPC_Snow (공통 파라미터)

- Blueprint와 여러 머티리얼이 **공용으로 읽는 값** 저장소.
- 눈 영역의 위치/크기, 오프셋, 감쇠, 강도 같은 **Vector/Scalar 값**을 담는 용도로 보인다. 🔶
- 텍스처(RT) 자체는 MPC에 넣을 수 없다. RT는 머티리얼 **텍스처 파라미터**로 전달한다.

| 전달할 데이터               | 전달 방식                                                         |
| --------------------- | ------------------------------------------------------------- |
| 위치 오프셋, 영역 크기, 감쇠, 강도 | `MPC_Snow`의 Vector/Scalar Parameter                           |
| `RT_Snow` 같은 텍스처      | Texture Object Parameter + MID의 `Set Texture Parameter Value` |
| RT를 고정 참조할 때          | 머티리얼 Texture Sample에 RT 에셋 직접 지정                              |

> 파라미터 이름은 **철자까지 정확히 일치**해야 한다. 머티리얼은 `SnowRT`인데 Blueprint가 `RT_Snow`라는 이름으로 설정하면 아무 일도 일어나지 않는다. (MPC도 동일)

---

## 6. M_Snow와 MF_DeformationProcess (중요 정정)

초기에는 "M_Snow 안에서 RT를 직접 샘플링한다"고 가정했지만, **실제로는 그렇지 않다.** 변형 처리 전체가 `MF_DeformationProcess` Material Function 안에 들어 있다.

### 6.1 M_Snow에서 확인된 것 ✅

- 프로젝트 마스터 머티리얼. Surface / Opaque / Default Lit
- `M_Snow_Inst` 라는 머티리얼 인스턴스도 존재
- `Texture Object` 노드가 `MF_DeformationProcess`의 `In (T2d)`로 들어간다.

### 6.2 MF_DeformationProcess의 입출력 ✅
**입력**

| 핀                         | 타입                   | 설명                        |
| ------------------------- | -------------------- | ------------------------- |
| `In`                      | T2d (Texture Object) | 변형 RT (`RT_Snow`가 들어올 자리) |
| `OriginalNormal`          | V3                   | 기존 노멀 계산 결과               |
| `Use Nanite Tessellation` | SB (Static Bool)     | Nanite 분기 스위치             |

**출력**

| 핀                       | M_Snow에서 연결되는 곳         |
| ----------------------- | ----------------------- |
| `SnowDeformationOffset` | World Position Offset ✅ |
| `SnowDeformationNormal` | Normal ✅                |
| `SnowTrailMask`         | 트레일 관련 계산 🔶            |
| `SnowTrackedMask`       | 밟힌 영역 마스크 🔶            |
| `SnowRidgeMask`         | 가장자리 융기 🔶              |
| `NaniteDisplacement`    | Nanite 변위 🔶            |

### 6.3 이식 시 경계선

```
RT_Snow (Texture2D)
   │
   ▼
Texture Object Parameter ("SnowRT" 등)
   │
   ▼
MF_DeformationProcess.In
   ├─ 월드 위치 → RT UV 변환 ❓ (함수 내부, 미판독)
   ├─ RT 샘플 및 변형 계산  ❓
   └─ 출력 6개로 결과 반환
   │
   ▼
M_Snow가 출력별로 연결 (WPO / Normal / Mask ...)
```

`MF_DeformationProcess` **내부**의 노드 구성과 UV 계산은 아직 끝까지 확인되지 않았다. 이식 성공 여부는 대부분 이 부분에 달려 있다.

---

## 7. 좌표 정합 (가장 흔한 실패 원인)

RT에 **그리는 좌표**와 머티리얼이 **읽는 좌표**가 같은 월드 영역을 가리켜야 한다.

```
눈 표면의 World Position XY
   │
   ├─ (Generator가 관리하는 영역 원점/오프셋) 을 뺌
   ├─ (눈 영역의 월드 크기) 로 나눔
   ▼
0~1 범위 UV  →  RT_Snow 샘플
```

반드시 맞춰야 하는 값:

- Generator가 RT에 그리는 **중심 위치**
- 머티리얼이 사용하는 **월드 영역 원점**
- **월드 영역 크기**
- XY 축 방향과 텍스처 **V축 방향** (뒤집힘 여부)
- Generator의 오프셋 갱신과 머티리얼 파라미터 갱신 **시점** (한 프레임 어긋나면 자국이 밀린다)

증상별 진단:

| 증상                  | 의심 원인                    |
| ------------------- | ------------------------ |
| 자국이 전혀 안 보임         | RT 미전달, 파라미터 이름 불일치      |
| 자국이 밀려서 보임          | 원점/오프셋 불일치, 갱신 시점 차이     |
| 자국이 늘어나거나 줄어 보임     | 영역 크기 불일치                |
| 상하/좌우 반전            | 축 방향, V축 뒤집힘             |
| 흑백 마스크는 보이는데 변형이 없음 | 샘플값이 WPO/Normal까지 연결 안 됨 |

---

## 8. 직접 만들 때 권장 순서

한 번에 전부 만들지 말고, **단계마다 눈으로 검증**하면서 진행한다.
### 단계 1: RT 전달부터 검증

1. `RT_Snow`(2048², RGBA16f) 생성
2. 눈 Plane에 **Dynamic Material Instance** 생성
3. `Set Texture Parameter Value("SnowRT", RT_Snow)`
4. 머티리얼에서 Texture Object Parameter → 샘플 → **Base Color에 바로 연결**
5. RT에 임시로 흰 점을 그려서 눈 표면에 보이는지 확인
    

### 단계 2: 좌표 정합

1. `MPC_Snow`에 영역 원점/크기 파라미터 추가
2. 월드 위치 → UV 변환 구현
3. 알려진 위치에 그린 점이 정확히 그 위치에 뜨는지 확인
    

### 단계 3: 자국 캡처

1. `SceneCaptureComponent2D`로 대상 Actor를 `RT_Capture`에 캡처
2. Trace Component가 `FCustomTrace`를 만들어 Generator에 전달
    

### 단계 4: 누적과 복원

1. `RT_Capture`를 `RT_Snow`에 합성하는 `M_Merge` 작성
2. `CopyToHistory` 구현
3. 감쇠(`SetSnowAttenuation`) 추가로 복원 확인
    

### 단계 5: 변형 표현

1. 샘플값 → World Position Offset, Normal
2. 마스크(Trail/Tracked/Ridge)로 가장자리 융기 추가
3. 필요 시 Nanite Displacement
    

---

## 9. C++ 이식 메모 (RTStruct.h)

```
Blueprint FCustomTrace
        ↓ 필드 이름·타입·의미 대응
C++ RTStruct.h 구조체
        ↓
C++ Generator가 구조체를 받아 처리
```

구조체 선언만 옮기면 **로직은 옮겨진 것이 아니다.** 아래 요소가 함께 구현되어야 결과가 같다.

- 요청을 **배열에 추가하는 시점**
- 씬 캡처를 **수행하는 시점**
- RT를 **Clear하고 다시 그리는 순서**
- History 복사 시점 (프레임마다인지, 주기적인지)
    

---

## 10. 확정 / 추정 / 미확인 정리

### ✅ 확정

- Trace Component → `FCustomTrace` 생성 → Generator 등록
- Generator의 `SceneCaptureComponent2D` 구조, 함수 목록, `RTResolution` 2048
- `RT_Capture` 256² R16f, `RT_Intermediate` 2048² RGBA16f
- `CopyToHistory`가 `RT_Snow`를 입력으로 읽음
- `MergeRT`에서 `RT_Snow`를 Clear, `M_Merge`에 Vector Parameter 설정
- `M_Snow`가 `MF_DeformationProcess`를 호출, 입출력 핀 구성, `WPO`/`Normal` 연결
    

### 🔶 추정

- `RT_Capture` → `RT_Intermediate` → `RT_Snow` 순서
- `CopyToHistory`의 대상이 `RT_SnowHistory`
- 감쇠가 복원 효과를 만든다는 해석
- Mask 출력들의 정확한 용도
    

### ❓ 미확인 (다음에 확인할 것)

1. `MF_DeformationProcess` 내부: 어떤 채널을 읽는지, 월드→UV 계산 방식
2. `MergeRT`의 Draw 노드 순서 (어떤 텍스처를 어떤 RT에 그리는지)
3. Generator가 눈 Plane의 머티리얼에 RT/MPC를 **실제로 세팅하는 노드**
4. `M_Snow_Inst`의 텍스처 파라미터 값과 Generator가 쓰는 이름 일치 여부
5. `RT_Intermediate`가 정확히 무엇을 위한 것인지 (합성 스크래치 / 블러 / falloff)
6. `FFootPrintInfo` 필드와 FootTrace의 실제 전송 경로
7. Tick 순서 (캡처 → 합성 → 복사 → 감쇠가 프레임 내 어느 순서인지)
    

### Codex에게 그대로 물어볼 질문 예시

```
MF_DeformationProcess 내부를 열어서 다음을 정리해줘.
1) In(T2d)에서 어떤 UV로 샘플하는지 (월드 위치 → UV 노드 체인)
2) 샘플의 R/G/B/A 채널이 각각 어떤 출력(Offset, Normal, Mask)에 쓰이는지
3) MPC_Snow의 어떤 파라미터를 읽는지 (이름 그대로)

BP_SnowGenerator의 MergeRT 함수는
Draw Material to Render Target 노드마다 [대상 RT, 사용 머티리얼, 입력 텍스처]를
실행 순서대로 표로 정리해줘.
```

---

## 11. 자주 막히는 지점 체크리스트

- Generator가 맵에 배치되어 있고, 컴포넌트 `BeginPlay`에서 참조를 찾는가
- Trace Component가 실제 자국을 남길 Actor에 붙어 있는가
- 이동량/속도 조건을 통과해서 `FCustomTrace`가 생성되는가
- `RT_Capture`에 자국이 찍히는가 (RT 미리보기로 확인)
- `RT_Snow`가 매 프레임 갱신되는가
    
- 머티리얼 텍스처 파라미터 이름과 Blueprint의 `Set Texture Parameter Value` 이름이 **철자까지** 같은가
- 눈 Plane에 **MID를 만들어서** 머티리얼로 적용했는가 (MID 없이 파라미터를 세팅하면 안 보인다)
- MPC 영역 원점/크기가 Generator와 머티리얼에서 같은 값인가
- RT 포맷이 정밀도를 지원하는가 (R16f / RGBA16f)
- `MF_DeformationProcess` 출력이 M_Snow의 WPO/Normal에 연결되어 있는가
    

---

_이 문서는 에디터에서 확인된 사실과 추정을 구분해 작성되었습니다. ❓ 항목은_ `_MF_DeformationProcess_`_와_ `_MergeRT_`_를 직접 열어 확인한 뒤 갱신하세요._