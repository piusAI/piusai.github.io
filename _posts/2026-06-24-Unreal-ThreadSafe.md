---
layout: post
published: true
title: Unreal DataAsset vs PrimaryDataAsset
subtitle: Constraint
date: 2026-06-25 13:42:00 +0900
description: Unreal Code
categories:
  - GameEngine
tags:
  - UnrealClass
  - Unreal
---

Thread Safe가 필요한 이유와 C++에서의 구현?

이도 마찬가지로, DataDriven할 수있도록 최신 구조들이 변형 되고있을때 꼭 필요한 데이터 정리 기법중 하나이다.
``` cpp
UCLASS()
class UMyAnimInstance : public UAnimInstance
{
	GENERATED_BODY()
public :
	virtual void NativeThreadSafeUpdateAnimation(float DeltaSeconds)
};

void UMyAnimInstance::NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
	//자동 Worker Thread
	//PropertyAccess 없이 직접 멤버 변수 읽기 가능
}

```
- ThreadSafe가 아닌 함수(Ex, Update) : Game Logic(메인 스레드)에서 실시간으로 변하는 변수들을 마구잡이로 읽고 쓴다. 만약 엔진에 Animation을 연산할때, 로직쪽에서 데이터를 수정중이라면 데이터가 튀거나(Race Condition), 런타임 에러가 발생 할 수도 있다.
- Thread Safe 함수(사용 중인 함수): "지금부터 Animation 계산을 시작할테니, 이 데이터들은 수정하지마!"라는 일종 잠금(Read-Only SnapShot)상태에서 연산을 수행.
  ->Game Logic이 뒤에서 Data를 바꾸든 말든 Animation 연산이 안정적으로 계산을 끝낼 수 있다.


---
### Unrealengine 에서의 THread Safe

> Game Thread (캐릭터 이동 -**쓰기**) : Velocity Update
> Worker Thread (애님 계산 -**읽기**) : Velocity 읽고 blending 계산

이 둘이 동시에 들어간다면 ? ?
>Game Thread : velocity Z:500 
>Worker Thread : 0??? 500??


### PropertyAcess

Game Thread 값 업데이트 완료 -> PropertyAccess가 복사본 만들어 넘겨줌 -> Worker Thread는 복사본만 읽음
원본 건드리지 않고, **스냅샷 복사본** 넘겨주는 방식

#### 일반 방식 (ThreadSafe 안함)


> Worker Thread에서 **TryGetPawnOwner()->GetVelocity()** 직접 호출
> Game Thread가 동시에 Pawn 데이터 수정중일 수 있음
> -> Crash or 쓰다만 값 읽힘


#### PropertyAccess 방식
> GameThread끝날때
> "지금 Pawn의 Velocity 값 복사해둘게" : 스냅샷 저장
> Worker Thread는 그 복사 값만 읽음
> -> 원본 건드리지 않으니 충돌 X


경로지정과 ThreadSafe를 보장하는 Property Access!