---
layout: post
title: Thor Hammer DevLog (DX11)
date: 2026-06-07 16:02:00 +0900
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

Thor DX 프로젝트 DevLOG

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

![FrameWork_001]({{ 'assets/postimg/ThorPRJ/FrameWork_001.jpg' | relative_url }})
 삼각형을 그린 프레임웍이다. 위 프레임워크에서 시작을 했다.


어느정도 필요한 모듈들은 생각해두었다.
- TextureClass, DDSTextureLoad, Input Module, CameraClass Module, 등등등...

머리속에 어떤식으로 쫙불러오면 좋겠다 라는 정도의 구상도까지는 그려지지 않기에 7일동안 제작을 하면서 조금씩 변경하고 할 계획이다.

**Core Dir :** 수업 중에 활용했던 프레임워크, Rastertek, Braynzar의 코드들을 십분 활용하여 ***모듈식***으로 붙히는 방향으로 진행 한다.

![Result01]({{ 'assets/postimg/ThorPRJ/Result01.jpg' | relative_url }})

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



## 💻 Day 2. 핵심 모듈 뜯어 붙히기!

MeshScaling : Hammer Size 15배

```
GraphicsClass::Initialize(int screenWidth, int screenHeight, HWND hwnd)
{
	...
	m_Camera->SetPosition(0.0f, 0.0f, -5.0f);
	...
}
```
Dx Camera location : (0,0,-5)

![HoudiniHammer]({{ 'assets/postimg/ThorPRJ/DxSceneTest.png' | relative_url }})


|구분|**DirectX**|**Houdini**|
| :--- | :--- | :--- |
|좌표계**(Coordinate)**|**왼손 좌표계** (Left-handed)|**오른손 좌표계** (Right-handed)|
|Z축 방향|화면 **안쪽** : $+Z$<br>(모니터 내부)|화면 **바깥쪽** : $+Z$<br>(사용자 시선)|
|원점을 바라보기 위한 Camera|$(0, 0, -5)$| $(0, 0, -5)$|
| **Object Unit WorldScale** | <td colspan="2" style="text-align: center;">DX : Houdini = 1:1 Size 매칭</td> |

DX : Houdini = 1 : 1 Unit 대응 한번 더 확인


**Data Driven** : designer/ artist가 데이터에 의해 작업 가능하게 하는 데이터 중심 설계
Data Driven까지 신경써서 Outlineer, Transform 계층구조를 만드는 것까지는 못함

추가 모듈
- Camera 가져와서 먼저 가볍게 이동할 수 있도록 넣어주기
- VS로의 World position 위치 이동, 디자이너가 찍는 것 처럼 5초 이후 이동 할수있도록 queue로 처리..?


###  Module 201 Camera

```
void CameraClass::GetViewMatrix(XMMATRIX& viewMatrix)
{
	viewMatrix = m_viewMatrix;
}

bool GraphicsClass::Render()
{
	...
	m_Direct3D->GetWorldMatrix(worldMatrix);
	m_Camera->GetViewMatrix(viewMatrix);
	m_Camera->GetCamView(camMatrix); //02 camview matrix 추가
	m_Direct3D->GetProjectionMatrix(projectionMatrix);
	...
	
	result = m_TextureShader->Render(m_Direct3D->GetDeviceContext(), m_Model->GetIndexCount(),
	worldMatrix, viewMatrix, projectionMatrix);
}
```


Get하는게 여기서 return하는것이 아니라 Class 내부에 있는 m_viewMatrix를 받아서 GraphicsClass에서 활용하기위함임!


[이식중 통신]
Braynzar의 코드는 이식성 쉽지않게 되어있음.

* 01 InputClass <--> CameraClass 통신
InputClass:: moveRight<-->CameraClass::UpdateCamera ***통신***
CameraClass::CamYaw<-->InputClass::CamYaw ***통신***
-> camYaw, camPitch, moveRight, MoveBackForward를 새로운 InputState.h로 Struct로 활용

Camera정보로써 남겨둘 것이어서, Camera객체 생성자에서 new로 만들고 소멸자에서 사라지게 만들었고,
InputState를 Struct로써 DetectInput에서 활용할 수있도록 받음!


![UpdateCam]({{ 'assets/postimg/ThorPRJ/UpdateCameraFramework.png' | relative_url }})

Update Camera를 Input못하고있어서
SystemClass안에서의 Graphic instance -> updateCamera();


![DXLog]({{ 'assets/postimg/ThorPRJ/DXCameraLOG2.png' | relative_url }})

Compile은 됨! DIKeyboard nullptr에러말고 
InitDirectInput이식 안했어서 DIKeyboard가 nullptr이었음.

![HINSTANCE]({{ 'assets/postimg/ThorPRJ/HInstancetransplantation.png' | relative_url }})
추가 HInstance를 받기위해서 InputClass::InitDirectInput으로 받아줬음.

<p align="center">
  <img src="{{ 'assets/postimg/ThorPRJ/Thor_MouseMove.gif' | relative_url }}" alt="MouseMove">
</p>
* ***Mouse Move GIF*** : Mouse PitchYaw 이식완료

<p align="center">
  <img src="{{ 'assets/postimg/ThorPRJ/Thor_KeyboardMove.gif' | relative_url }}" alt="KeyboardMove">
</p>
* ***KeyBoard GIF*** : WASD FreeLookCamera(FLC) 이식 완료



## 💻 Day 3. Click 위치로 이동시키기!

### Navigation 추가

- 간단하게 먼저 UE처럼 E를 눌렀을때 위아래로 이동하도록 손봐주었다.
	- 01 InputState struct의 moveupdown추가
	- 02 moveupdown관련 함수 추가
	- 03 Virtual Keyboard 등록!
![[assets/postimg/ThorPRJ/Thor_KeyboardUpdown.gif]]

<p align="center">
  <img src="{{ 'assets/postimg/ThorPRJ/Thor_KeyboardUpdown.gif' | relative_url }}" alt="KeyboardMove">
</p>




### Object Model Texture

![[assets/postimg/ThorPRJ/Thor_Dollyinout.gif]]

<p align="center">
  <img src="{{ 'assets/postimg/ThorPRJ/Thor_Dollyinout.gif' | relative_url }}" alt="KeyboardMove">
</p>
01 DDSTextureLoader 이식
02 관련 Texture Class 함수, initialize 확인
03 Sementic 확인

```
struct VertexInputType
{
	float4 position : POSITION;
	float2 tex : TEXCOORD0;
}

struct PixelInputType
{
	float4 position : SV_POSITION;
	float2 tex : TEXCOORD0;
}

PixelInputType TextureVertexShader(VertexInput input)
...

TexturePixelShader(PixelInputType input) : SV_TARGET
```

{% highlight cpp %}
D3DCompileFomrFile(...,"TextureVertexShader", ...)
D3DCompileFomrFile(...,"TexturePixelShader", ...)
{% endhighlight %}

04 `D3D11_INPUT_ELEMENT_DESC polygonLayout[2]` 확인
color 정보를 받던 float4 ->uv 정보를 받는 float2로 구조체 수정

{% highlight cpp %}
...
polygonLayout[1].Format = DXGI_FORMAT_R32G32_FLOAT;
...
{% endhighlight %}


```
$p=(Resolve-Path 'Colorps.hlsl').Path; $c=Get-Content $p -Raw; [System.IO.File]::WriteAllText($p,$c,(New-Object System.Text.UTF8Encoding($false)))
```
Shader 파일 `.vs, .ps` 에러시 억지로 인코딩



### Actor Set-up

![FrameWork_003]({{ 'assets/postimg/ThorPRJ/FrameWork_003.png' | relative_url }})
수정 전 FrameWork

![FrameWork_003]({{ 'assets/postimg/ThorPRJ/FrameWork_0031.png' | relative_url }})
수정 후 FrameWork


일단 CameraClass는 월드당 하나로 제작하도록 해놓았음.
### 다중 Object Import

![FrameWork_003]({{ 'assets/postimg/ThorPRJ/MultiUpdate.png' | relative_url }})

: vector, 동적배열로 Object import / Spawn 할 수 있도록 구현

![FrameWork_003]({{ 'assets/postimg/ThorPRJ/InitializeBuffer.png' | relative_url }})
: Initialize Buffer에다 Model World position offset 들어갈 수 있도록 구현

-> 따로 개별적으로 위치를 주기보다, Model 생성자 에다가 설정할 수있도록 잡아줌

![FrameWork_003]({{ 'assets/postimg/ThorPRJ/VtxMove.png' | relative_url }})
-> 하지만 Vtx를 직접 이동했기 때문에 위 이미지처러 Dcc art에서의 Component(점·선·면)로 이동 한 느낌.



##### 추가 Transform component로 붙히기

*** DIR:*** 추후 model 생성할때 하나씩 initialize 잡는것도 좋지만, Transform hiearchy component들어갈 수있도록 잡아주면 좋을 것이라 생각



![[assets/postimg/ThorPRJ/TransformComponent.png]]

![Render Pipeline Image]({{ 'assets/postimg/ThorPRJ/TransformComponent.png' | relative_url }})

Transform Component Struct를 만들어서 Model에 붙혀놓았는데, model Class 생성자에서 위치를 조정하지 않고, `TransformComponent::SetTransform()`으로 잡음

HLSL, GPU단에서 수정한다면, ModelClass 개별당 설정하기 쉽지 않을 것 같아
**VS**시, 모델 개별당 World Matrix -> `이곳에 추가` ->View Matrix로 행렬 곱으로 들어가면 될 것이라 판단.
`WorldClass::SetShaderParameter`에서 TRS matrix를 넣어서
***GPUShaderClass***에서 SetShaderParameter의 인자로 받을 수 있도록 통신한 이후, TRS matrix를 VS에서 Matrix순서를 RST로 곱해줌.


![[assets/postimg/ThorPRJ/WorldPositionOffset.png]]

![WorldPositionOffset]({{ 'assets/postimg/ThorPRJ/WorldPositionOffset.png' | relative_url }})
: 갑자기 분위기 WorldPosition offset처럼들어감.
-> 행렬 곱 순서가 잘못되었다 생각했지만,

![[TransformComponentDone.png]]

![TransformComponentDone]({{ 'assets/postimg/ThorPRJ/TransformComponentDone.png' | relative_url }})
: `XMMatrixTranspose(TransformMatrix)`로 전치로 만들어주어야함. `TransformComponent::Matrix`(GPU를 위한 행렬) 이라, HLSL에서 와 CPU에서 행렬곱으로 하는 matrix가 다르기 때문. 


![[FrameWork0032_Transform.png]]
![FrameWork0032_Transform]({{ 'assets/postimg/ThorPRJ/FrameWork0032_Transform.png' | relative_url }})
: Transform Component의 FrameWork

---
[현재 진행]

### Level Design With Houdini
지금까지의 구현으로 Object 배치 가능,
먼저 World 위치를 Houdini로 Model 배치를 한 이후에 다시 Dx로 가져올 계획
-> 이후 IA에서 instancing도 넣을 계획. 어차피 위치는 맞춰야하니까


#### Houdini Quternion을 활용한 랜덤한 회전값


![[Quternion_Rot.gif]]
![Quternion_Rot]({{ 'assets/postimg/ThorPRJ/Quternion_Rot.gif' | relative_url }})



{% highlight hlsl %}
vector up = set(0,1,0);
float angle=radians(chf('rotate'));
vector4 q = quaternion(angle * fit01(rand(@ptnum+chf("seed")), chf('min'), chf('max')), up);
@orient = q;
{% endhighlight %}


![[LevelDesign.png]]
![LevelDesign]({{ 'assets/postimg/ThorPRJ/LevelDesign.png' | relative_url }})
Level Design이라고 하기도 미안한 배경 prob 배치 완료



![[semiHDRI.png]]
![semiHDRI]({{ 'assets/postimg/ThorPRJ/semiHDRI.png' | relative_url }})
: HDRI sphere를 넣지는 못하지만,, 배경이 너무 심심해서
- Maya SkyDome형식으로 제작!


![[NOTyetTriangle.png]]
![NOTyetTriangle]({{ 'assets/postimg/ThorPRJ/NOTyetTriangle.png' | relative_url }})

아직 Triangle화를 시키지 않았기 때문에 이렇게 배경녹색이 나옴.
UE에 던지자마자 삼각형으로 만드는 이유가 이 때문

DX에서는 항상 Vtxbuffer->indexbuffer로 만들때 삼각형으로!
그 구조체 타입은 RasterDesc.TrianglePan 등등으로 지정했었다!

![[likeHDRI.gif]]

![likeHDRI]({{ 'assets/postimg/ThorPRJ/likeHDRI.gif' | relative_url }})
[금일 최종 작업 HDRI처럼 만든 형태]
: worldPosition (0,0,0)을 향하도록 Normal 방향 바꿔 UV - Texture만 넣어줌




## 💻 Day 4.  Mouse on/off & Screen Postion


##### 01 Tilt On/Off
현재 Moust Input이 바로 화면 Tilt로 들어가고 있음.
-> Tilt On/Off로 Spacebar 누르면서 해야지 화면 Tilt 할수있도록!
`DIMOUSESTATE`에서 lX, lY로만으로 마우스 로직 처리말고, Spacebar도 16진법 
Bit Masking -AND Masking 추가해줌

{% highlight cpp %}
mouseCurrState.lX != mouseLastState.lX) && (keyboardState[DIK_SPACE] & 0x80 )
{% endhighlight %}

![[HoudiniCamSetup.png]]
![HoudiniCamSetup]({{ 'assets/postimg/ThorPRJ/HoudiniCamSetup.png' | relative_url }})
: Init 카메라 위치 추가


![[HoudiniImport.png]]
![HoudiniImport]({{ 'assets/postimg/ThorPRJ/HoudiniImport.png' | relative_url }})
: 카메라 축방향이 Houdini Z(모니터 앞 방향), DX(모니터 안 방향)
이라서 그냥 수치로 잡아줌
![[DXImport.png]]
![DXImport]({{ 'assets/postimg/ThorPRJ/DXImport.png' | relative_url }})
: Houdini(WorldScale : 7.5) -> Blender -> Dx 해줌

##### 02 Mouse Show
`WNDCLASSEX` 에서 수정 인줄알았는데
Input 들어가면서 마우스가 없어진듯


`IDirectInputDevice8::Acquire` : InputDevice 접근 권한 확인

![[MouseOnoff.gif]]
![MouseOnoff]({{ 'assets/postimg/ThorPRJ/MouseOnoff.gif' | relative_url }})

:마우스 On/off

{% highlight cpp %}
if (mouseCurrState.rgbButtons[0]) //LMB
{
	// SetCursor(LoadCursor(NULL, IDC_ARROW));
	ShowCursor(true);
}
{% endhighlight %}


##### 03 Time Module
![[TimeModule.png]]
![TimeModule]({{ 'assets/postimg/ThorPRJ/TimeModule.png' | relative_url }})

text module로 profile 되고 있는것들 이식!

FPS, CPU 확인 profiler

`ID3D11DepthStencilState`  Disable / able 로 따로 구조체 개별로 만들어서 switch해줘야함!
text올릴때는 Zbuffer를 끄고(DepthStencilState off 해야함!)!

`ID3D11BlendState`도 마찬가지로 AlphaBlend를 on/off 하는 구조체

//화면 facing되도록
XMMATRIX baseViewMatrix;
...
m_Camera->GetViewMatrix(baseViewMatrix);
...
result = m_Text->Initialize(... baseViewMatrix);
`
Text ui로 나올수있도록 화면 Facing하게 해야함!
이렇게 해놓고 여기가 문제였음..


```
...

float4 FontPixelShader(PixelInputType input) : SV_TARGET
{
    float4 color;
    color = shaderTexture.Sample(SampleType, input.tex);
    
    if (color.r == 0.0f)
    {
        color.a = 0.0f;
    }
    else
    {
        color.rgb = pixelColor.rgb;
        color.a = 1.0f;
    }
    return color;
}
```

: alpha를 red채널로 확인해서 0이면 alpha 0으로 받도록 ps를 잡음.

%%  언리얼에서 그토록 외치는 VS와 PS중에서 VS가 가볍다하는 이유를
여기 Dx에서 몇번이고 hlsl을 쳐보다보니 이해가 감.
vs는 준비된 행렬로 한번만 곱하면 되지만
ps는 pixel당 곱해지기에 무거운거였군! %%

![[Framework004_time.png]]
![Framework004_time]({{ 'assets/postimg/ThorPRJ/Framework004_time.png' | relative_url }})

Framework 이식 성공 Framework
: painters 알고리즘
DepthStencilBuffer ON, AlphaBlend Off -> 3D Object 렌더링 -> DepthStencilBuffer Off, AlphaBlend ON -> 2D UI Text Alpha제거

![[FPSProfiler.png]]
![FPSProfiler]({{ 'assets/postimg/ThorPRJ/FPSProfiler.png' | relative_url }})


{% highlight cpp %}
//GraphicsClass::Initialize()
//m_Camera->GetViewMatrix(baseViewMatrix);

// 2D 텍스트 렌더링에 사용되는 baseViewMatrix는 카메라가 이동하거나 회전하지 않은 기본 상태(0, 0, -1에서 원점을 바라봄)함
// m_Camera의 회전 및 이동 값이 적용된 뷰 행렬을 사용하면 2D 텍스트가 화면 밖으로 밀려나 보이지 않게 됩니다.
XMVECTOR eyePosition = XMVectorSet(0.0f, 0.0f, -1.0f, 0.0f);
XMVECTOR focusPoint = XMVectorSet(0.0f, 0.0f, 0.0f, 0.0f);
XMVECTOR upDirection = XMVectorSet(0.0f, 1.0f, 0.0f, 0.0f);
baseViewMatrix = XMMatrixLookAtLH(eyePosition, focusPoint, upDirection);
{% endhighlight %}

컴파일이랑 다되었는데 3시간정도 못찾아서
행렬 연산 때문인지 2D UI가 안나와서 ai 돌려서 확인.


3D 메인 카메라 객체인 `m_Camera`로부터 뷰 행렬을 덮어씌우는 대신, 2D 텍스트 렌더링을 위해 항상 정면을 고수하는 고정된 뷰 행렬(`XMMatrixLookAtLH`)을 직접 생성하여 `baseViewMatrix`에 넘겨주도록 수정.
~~Gemini 사랑해~~

##### 04 Click Input
Screen 위치 받기!

```
	POINT cursorPos;
	GetCursorPos(&cursorPos);
	ScreenToClient(hwnd, &cursorPos);
	
	bool currLMB = mouseCurrState.rgbButtons[0] & 0x80;

	if (!prevLMB && currLMB) //LMB
	{
		//SetCursor(LoadCursor(NULL, IDC_ARROW));
		ShowCursor(true);
		
		OutputDebugStringW(L"Left Button\n");
		wchar_t buf[256];
		swprintf_s(buf, L"ClickX : %d, ClickY : %d \n", cursorPos.x, cursorPos.y);
		OutputDebugStringW(buf);
	}
```
![[ScreenPositionLOG.png]]
![ProceduralBridge]({{ 'assets/postimg/ThorPRJ/ProceduralBridge.png' | relative_url }})
: Screen Position 찍음

![[ProceduralFense.gif]]
![ProceduralFense]({{ 'assets/postimg/ThorPRJ/ProceduralFense.gif' | relative_url }})
: Copy to point로는 Orient까지 맞춰서 Dx 툴만드는데 오래걸리니 이 Mesh는 IA에서 Instancing으로 처리안할것.

-> BBX확인후 segmentation 길이따라 다리 설치.

![[DXSetupDone.png]]
![DXSetupDone]({{ 'assets/postimg/ThorPRJ/DXSetupDone.png' | relative_url }})
* 01 Model LevelDesign - Model 정적 생성
Fense 전체 하나의 Obj Import

정적 Mesh 생성 완료

## 💻 Day 5.  screen to World position !

### 01 Hammer 위치 이동
A. Screen to World Position
B. World Position Que로 이동 순서 Data Structure 생성
C. Hammer 이동 구현

##### A. Hammer 위치 이동

* screen Posiiton, WorldPosition Hashmap
이번에는 Hashmap을 활용해서 뽑음
```
//define
unordered_map<string, float> screenPosition;

//initialize
screenPosition.insert(make_pair("x", 0));
..

//input
POINT cursorPos;
GetCursorPos(&cursorPos);
ScreenToClient(hwnd, &cursorPos);

screenPosition["x"] = cursorPos.x;
screenPosition["y"] = cursorPos.y;
...
wchar_t buf2[256];
swprintf_s(buf2,L"HashClick:  X : %f, Y : %f \n", screenPosition["x"], screenPosition["y"]);
OutputDebugStringW(buf2);

```

![[hashLOG.png]]
![hashLOG]({{ 'assets/postimg/ThorPRJ/hashLOG.png' | relative_url }})

: screenXY 받음

* Screen to World Position
현재 Screen Position을 받고있음
UE에서의 `DeprojectScreenToWorld()`참고
```
bool APlayerContoller::DeprojectScreenToWorld
(float screenX, float screenY, vector& WorldLocation, vector& WorldDirection)

```

UE는 output Parameter로 제작 해놓음
```
FMatrix ComputeViewProjectionMatrix() const
	{
		return FTranslationMatrix(-ViewOrigin) * ViewRotationMatrix * ProjectionMatrix;
	}
```
: Unreal engine Method도 역시 이렇게 역변환으로 돌렸다
*  screen to world position VS의 행렬 변환을 역변환을 거쳐서
// VS가 World -> view -> projection
// projection inverseMatrix -> view inverseMatrix -> world Matrix to World Vector



```
FSceneViewProjectionData ProjectionData;
		if (LP->GetProjectionData(LP->ViewportClient->Viewport, /*out*/ ProjectionData))
		{
			FMatrix const InvViewProjMatrix = ProjectionData.ComputeViewProjectionMatrix().InverseFast();
			FSceneView::DeprojectScreenToWorld(ScreenPosition, ProjectionData.GetConstrainedViewRect(), InvViewProjMatrix, /*out*/ WorldPosition, /*out*/ WorldDirection);
			return true;
		}
```
grphics에 있는 Matrix Worldmatrix, viewmatrix, Orthomatrix 받아와서 돌림!


```

unordered_map<string, float> InputClass::DeprojectScreenToWorld(float screenWidth, float screenHeight, float screenX, float screenY)
{

// 01 Screen -> NDC (clip 공간)
float ndcX = (2.0f * screenX / screenWidth) - 1.0f;
float ndcY = 1.0f - (2.0f * screenY / screenHeight); // Y 반전!

// 02 NDC -> View (Projection 역행렬)
XMMATRIX invProj = XMMatrixInverse(nullptr, projectionMatrix);
XMVECTOR rayClip = XMVectorSet(ndcX, ndcY, 1.0f, 1.0f);
XMVECTOR rayView = XMVector3TransformCoord(rayClip, invProj);
//rayView = XMVectorSetZ(rayView, 1.0f); // 방향벡터 Z=1

// 03 View -> World (View 역행렬)
XMMATRIX invView = XMMatrixInverse(nullptr, viewMatrix);
XMVECTOR rayWorld = XMVector3TransformNormal(rayView, invView);
rayWorld = XMVector3Normalize(rayWorld);

// 04 Camera World Position
XMVECTOR rayOrigin = XMVector3TransformCoord(XMVectorZero(), invView);

// 05 Ray-Plane Intersection (Y = 5.0f 평면과의 교차점 계산)
// rayOrigin.y + t * rayWorld.y = 5.0f  =>  t = (5.0f - rayOrigin.y) / rayWorld.y
float originY = XMVectorGetY(rayOrigin);
float dirY = XMVectorGetY(rayWorld);
float targetY = 5.0f;

if (fabs(dirY) > 0.0001f)
{
	float t = (targetY - originY) / dirY;
	XMVECTOR intersection = XMVectorAdd(rayOrigin, XMVectorScale(rayWorld, t));

	WorldPosition["x"] = XMVectorGetX(intersection);
	WorldPosition["y"] = targetY; // Y축을 5.0f로 고정
	WorldPosition["z"] = XMVectorGetZ(intersection);
}
else
{
	// 평행한 경우 예외 처리
	WorldPosition["x"] = XMVectorGetX(rayOrigin);
	WorldPosition["y"] = targetY;
	WorldPosition["z"] = XMVectorGetZ(rayOrigin);
}

return WorldPosition;
}
```
: 행렬 돌리는 순서와 통신하는 아키텍쳐는 짰고, 
inverse /  view matrix 돌리는 것 헷갈려서 **AI 활용**


Screen -> NDC -> View -> World -> Camera -> Ray Plane

![[WorldPositionSet.gif]]
![WorldPositionSet]({{ 'assets/postimg/ThorPRJ/WorldPositionSet.gif' | relative_url }})
y축 고정으로 돌려놓음


* 02 input에 따른 Hammer 위치 변환 수정 - FSM Design




* 05 Input 위치 변형(Screen Position-> World Position 변형)

## 남은 작업



Queue와 같이 FIFO로 Click에 따른 datastructure 구성하기





* 04 Instancing 기법(IA 단계)
Instancing처리 : flag, OrnateHandle, Reliquary, Portal해야함


* 03 Hammer Object 자전
- Hierachy 구조 ( 빈 Object처럼 Transform 두번 감싸기..?)
Line @curveu 에 따라서 hammer Forward vector서서히 자전 멈추도록!


* 06 Camera 변경
  Play Draw Camera -> Play TPS(OTS) Camera 변형

* 07 2D Scene Click 가능하도록
  2D Title 이미지, Tutorial 이미지 생성 이후 click으로 다음넘기기
   Failed CutScene생성 (Rokki에게 발각되었습니다) , Complete Cut Scene : 아스가르드를 지킬 힘을 얻었습니다 (성공)


### 추가 고민
* 마우스 input 넣어놓고 이동시 wads로 왔다갔다 조금씩 할수있도록 할까..?
* ++ 충돌 algorithm..?


### 완료 사항
* Camera 키보드 위아래좌우 Move ( limit 걸어두기) 
* Mouse Input Rotation( Limit 걸기 )
* Object Texture 이식
* FPS, CPU 프로파일 이식 (Performance check)


[Rastertektriangle]: https://youtu.be/ZVBOs-fnr50?si=7jHpHkePuy9kL5IF