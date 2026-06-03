---
layout: post
title: DirectX 11 Framework Architecture 설계
date: 2026-06-02 16:02:00 +0900
description: 상용 엔진(Unreal)의 API 뒤에 숨겨진 렌더 파이프라인을 제어하기 위한 로우레벨 DirectX 엔진 프레임워크 설계 기록입니다.
thumbnail-img: /assets/postimg/ThorArchitecture/FrameWork_001.jpg
categories:
  - R&D
tags:
  - DirectX
  - Framework
  - Architecture
  - Graphics-Pipeline
---

Thor DX 프로젝트 LOG

설계라고하기에는 준비된 Module을 끼워맞춘다에 가깝지만, 그럼에도 불구하고 처음으로 저수준으로 프레임웍을 뜯어붙히고 다뤄보며 조합하는 Project이다.

대다수의 상용 엔진은 그래픽스 파이프라인을 고도로 추상화하여 제공을 한다. 엔진을 활용하며 고수준에서만 제작하다보면, 어떤 지점이 불편하고 편하고 알 수가 없다 생각한다. 직접 저수준에서 다뤄보는 작업이야말로 옳고 그름을 판단할 수있다는 생각은 AI 시대에서 뒤쳐졌다 생각할 지언정 틀린 방법은 아니라고 단언할 수있다. 탄탄히 쌓은 기본기만이 고수준의 작업이나, **CPU와 GPU 간의 데이터 흐름(Data Flow)**을 완벽하게 통제 할 수 있을 것이다.

본 포스팅에서는 상용 엔진 없이 C++와 로우레벨 그래픽스 API만을 활용해 구축한 **DirectX 게임 엔진 프레임워크의 구조적 설계**에 도전해볼 것이고, 핵심 모듈과 디자인 패턴도 적용해볼 계획이다.

RasterTek 삼각형 그리기까지 완료 한 이후, 모든 class를 이해하는 것은 시간적으로 불가능하고, 협업에 있어서도 모든 이해는 불필요하다는 결론을 내렸다.
지금 현재의 게임에 필요한 class만을 모듈식으로 붙히고, 물고 어디에서 사용하고 하는 형식의 프로젝트로 일주일동안의 로그를 찍어보려 한다.

Tip 두가지로는,
01 .hlsl shader 파일 : Build Exclude
02 Linker - Subsystem 설정
```
Windows (/SUBSYSTEM:WINDOWS)
```


---
이전의 삼각형까지의 이해는 [Rastertektriangle]에서 Summary로 볼 수있다. 영상에서처럼, 3주정도는 이해하는데에 많은 시간을 할애했다.

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/FrameWork_001.jpg' | relative_url }})
 삼각형을 그린 프레임웍이다. 위 프레임워크에서 시작을 했다.


어느정도 필요한 모듈들은 생각해두었다.
- TextureClass, DDSTextureLoad, Input Module, CameraClass Module, 등등등...

머리속에 어떤식으로 쫙불러오면 좋겠다 라는 정도의 구상도까지는 그려지지 않기에 7일동안 제작을 하면서 조금씩 변경하고 할 계획이다.

**Core Dir :** 수업 중에 활용했던 프레임워크, Rastertek, Braynzar의 코드들을 십분 활용하여 ***모듈식***으로 붙히는 방향으로 진행 한다.

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/Result01.jpg' | relative_url }})

---
## 🧩 Day 1. 모듈 뜯어붙히기

### Module101 TextureClass
001 Texture.h : Dx에서의 UV compile 방법은, DCC에서의 uv수정처럼 우아한 방식이 아니었다.
png 텍스쳐 파일을 엔진 Editor로 던지는 dds파일로 자동 변환해주는 것 부터 일이었다. VisualStudio 내부에서 dds변환(mipmap 생성도 가능)하는 일부터 시작했다.
(여기는 Rastertek Code를 보며 수정하면서 붙히는중..)

* .hlsl 파일 저장 에러

```
Unicode (UTF-8 without signature) - Code page 65001
```
위 파일로 저장하더라도 안된다면

```
powershell -Command "$p='~Colorvs.hlsl'; $c=Get-Content $p -Raw; [System.IO.File]::WriteAllText($p,$c,(New-Object System.Text.UTF8Encoding($false)))"
```
이렇게 아예 받아서 수정
  

[추가 / 수정 Class]
* **AlignedAllocationPolicy.h:** for warning C4316

* ApplicationClass -> **GraphicsClass** ,
  ColorShaderClass->**TextureShaderClass**
  TextureClass

하다가 갑자기 OBJ import 먼저 하고싶음

### Module102 OBJ import
002 modelclass.h : 역시나 Dx에서의 3D object import 방법은, Engine에서 던지는 형식이 아니었다. Maya/Houdini에서 .obj를 export할때 mtl은 그냥 재질의 찌꺼기이다라고 생각했지만,

![[assets/postimg/ThorPRJ/Hammer.jpg]]

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/Hammer.jpg' | relative_url }})
Pivot은 손잡이에 잡아두었다.

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/HammerObjDX.jpg' | relative_url }})
L : unit scale 1 Hammer Dx Import || R : Unit Scale 15 Hammer DX

물론 Dcctool에서 model scaling을 하면 좋겠지만 갑자기 HLSL으로 compile 해보고 싶었다.

GraphicsClass 내부에서 수정하는 행렬곱 scaling 으로는 CPU에서 처리하기에 병렬적 연산이 불가능하다 그래서 cbuffer vs로 수정해보고자 했다.
이전 과제에서는 CPU에서 처리했었지만 갑자기 분위기 VS!
* **01 Cbuffer 수정:**
{% highlight cpp %}
Colorvs.hlsl
cbuffer MatrixBuffer
{
	matrix worldMatrix;
	matrix viewMatrix;
	matrix projectionMatrix;
	matrix scaleMatrix; // Test Cbuffer ScalingTest
}
{% endhighlight %}

* **02 Texture class 수정: **

{% highlight cpp %}
  struct MatrixBufferType
  {
	  XMMATRIX world;
	  XMMATRIX view;
	  XMMATRIX projection;
	  XMMATRIX modelscale; // 네번째 인자로 컴파일
  }

  ...
  
  private:
  XMMATRIX scaleMatrix = XMMatrixScaling(15.0f, 15.0f, 15.0f);
  // 15배 정도의 ScaleMatrix 
  {% endhighlight %}


***03 최종 VS 수정**
```
  output.position = mul(input.position, worldMatrix);
  output.position = mul(output.position, scaleMatrix); //world에서 크기 scaleMatrix 곱
  output.position = mul(output.position, viewMatrix);
  output.position = mul(output.position, projectionMatrix);
```
  


 1일차 모듈 뜯어보기는 이 정도로 마무리 했다.

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/FrameWork_002.jpg' | relative_url }})

개발자들도 모듈을 뜯어 해킹하는 형식으로 많이 제작하고 있다 한다.
모든 코드를 이해하면 물론 좋겠지만 이해를 Trade-off하기로 했다. 7일 남은 시점에서는 교수님이 주신 코드와 Rastertek을 역추적하는 방식이어야한다.

진행하면서 게임 제작시 모든 분야의 코드를 이해하는 것은 불가능에 가깝다고 생각이 들기 시작했다.

---

  (현재 완료)

## 💻 Day 2. 핵심 모듈 뜯어 붙히기 완료

  

프레임워크의 심장인 핵심 게임 루프와 각 모듈의 초기화 라이프사이클을 담당하는 주요 구현체입니다. OOP 구조를 무분별하게 남용하기보다, 절차적인 데이터 흐름을 안전하게 보장하는 방식으로 설계했습니다.

  
  

[Rastertektriangle]: https://youtu.be/ZVBOs-fnr50?si=7jHpHkePuy9kL5IF