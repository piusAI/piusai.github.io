---
layout: post
title: Thor's Hammer 프로젝트 기획안[DX11]
date:   2026-05-29 15:32:00 +0900
description: C++의 OOP 구조를 탈피하여 고성능 그래픽스 렌더파이프라인 및 수리적 실시간 오브젝트 제어에 집중한 R&D 기획 및 연구 기록입니다.
img: Thor_Thumnail.png                  # 목록에 출력될 썸네일 이미지 (필요시 사용)
categories: [R&D]
tags: [DX11, Graphics, RenderPipeline, Framework, TrajectoryMath]
author: PIUS
---

<!-- 포스트 최상단 메인 아젠다 이미지 배치 -->

![Thor Architecture Agenda]({{ '/assets/postimg/ThorArchitecture/Thor_Thumnail.png' | relative_url }})
---

본 프로젝트는 생성형 AI 시대에 UE 단순 Function/기능 구현 위주의 작업에서 벗어나, 프레임워크 설계 능력과 그래픽스 저수준 제어 능력을 기르기 위해 진행된 **DX11 기반 실시간 오브젝트 비행 궤적 및 시스템 아키텍처 구현 프로젝트**입니다.

![Thor Architecture Agenda]({{ '/assets/postimg/ThorArchitecture/DXGraphics02.png' | relative_url }})

상용엔진(UE 등)에서의 무거운 상속 구조를 의도적으로 배제하고 수학 및 그래픽스 데이터 흐름에 집중하여 AAA급 스튜디오에서 요구하는 R&D 엔지니어링 역량을 확보하는 데 목적이 있습니다.

---

## 01. 코어 컨셉 (Core Concept)

### 1) 아스가르드 가상 훈련소 및 묠니르 비행 프로세스
![Thor Hammer Training Ground Core]({{ '/assets/postimg/ThorArchitecture/DXGraphics03.png' | relative_url }})

* **프로젝트 배경:** 토르의 월드인 아스가르드의 가상 훈련소를 배경으로 설정하여, 신화적 무기인 '묠니르(Mjolnir)'의 고유 비행 궤적 프로세스를 DirectX 11 환경에서 실시간 시뮬레이션 데모로 구현합니다.
* **아키텍처 지향점:** 렌더러와 시뮬레이션 로직 간의 커플링을 최소화하여 부드러운 카메라 네비게이션과 수학적 궤적 계산을 보장합니다.

### 2) 뷰포트 시점(Top-View / TPS)에 따른 네비게이션 시스템
![Hammer Navigation View]({{ '/assets/postimg/ThorArchitecture/DXGraphics04.png' | relative_url }})
위 참고 이미지 제작은 GPT 5.1 AI를 활용하였습니다.
* **Top View 마우스 입력 시스템:** 사용자가 탑뷰(Top-View) 시점에서 마우스 드래그를 통해 망치의 이동 위치 및 비행 경로를 자유롭게 드로잉(Draw Path)할 수 있도록 설계했습니다.
* **TPS 카메라 체이싱 알고리즘:** 비행 테스트가 시작되면 입구에서부터 TPS(3인칭 복합 시점) 카메라가 망치의 실시간 이동 궤적을 부드럽게 추적(Chasing)하며 아스가르드 훈련소 내부를 역동적으로 연출합니다.

---

## 02. 배경 Props 및 환경 구성

![Background Props Analysis]({{ '/assets/postimg/ThorArchitecture/DXGraphics05.png' | relative_url }})
위 3D mesh 생성은 Varco3d mesh 생성 Generation AI를 활용하였습니다.

* **Skybox 및 지형 렌더링:** 광활한 아스가르드의 밤하늘(은하수 패노라마) 이미지를 Sphere로 HDRI로 바인딩하여 고품질의 배경 스카이를 구축합니다.
* **환경 프롭 파이프라인:** 훈련소 내부를 구성하는 아스가르드식 깃발, 기둥, 포탈 게이트 등 핵심 기믹 프롭들을 독립적인 메시 인스턴스로 관리하여 그래픽스 리소스 메모리 적재량을 최적화합니다.

---

## 03. 시스템 아키텍처 오버뷰 (System Architecture Overview)

실시간 연산 최적화와 안정적인 트러블슈팅을 위해, 전체 그래픽스 데모의 핵심 모듈을 다음과 같은 논리 구조로 이원화하여 설계했습니다.

![System Architecture Specification]({{ '/assets/postimg/ThorArchitecture/DXGraphics06.png' | relative_url }})
먼저 확실히 구현 해야하는 기말과제와, 추가 도전 과제를 나누어 프로젝트를 진행하였습니다.

* **기말 과제 목표:** Graphics pipeline의 프레임워크 및 기본적인 카메라 기능을 먼저 구현

* **도전 과제 목표:** 충돌 알고리즘, PCG 라이트한 버전인 랜덤 배치 및 생성 로직, 베지어 곡선/보간법 적용 과같은 구현은 추후 작업 예정.


## 04. DX 11 커스텀 시스템 아키텍처

'게임 엔진 아키텍처(Game Engine Architecture)'에서 다루는 **게임 프레임워크 레이어와 엔진 코어 레이어의 명확한 분리 원칙**에 입각하여, '토르의 망치 연습소' 시뮬레이션을 구동하기 위한 전용 커스텀 아키텍처를 하단 구조와 같이 설계 및 구현했습니다.

내부적인 Win32 윈도우 래핑, 디바이스 및 디바이스 컨텍스트를 포함한 D3D11 초기화, 그래픽스 파이프라인의 구조체(Struct) 설계 등 저수준 백엔드 베이스는 [Rastertek][Rastertek]과 [BraynzarSoft][braynzar]의 아키텍처 모델을 철저히 분석하고 참고하여 내재화했습니다.

![System Architecture Specification]({{ '/assets/postimg/ThorArchitecture/FrameWork.png' | relative_url }})

(시스템 아키텍쳐 흐름도는 제작중)

### 🛠️ 아키텍처 코어 데이터 및 제어 흐름 (Architecture Flow)

본 커스텀 엔진 구조는 상용 엔진의 무거운 레거시(OOP 상속 구조 등)를 배제하고 오직 **'입력 처리 ➔ 상태 업데이트 ➔ 렌더 파이프라인'**의 고성능 단방향 데이터 흐름을 유지하는 데 집중했습니다.

* **Win32 & Core Engine Base:**
  시스템의 진입점(Entry Point)인 `SystemClass`가 Windows API 메세지 루프와 실행 흐름을 래핑하며, 내부의 `InputClass`와 `GraphicsClass`를 중앙 제어합니다. 이를 통해 하드웨어 입력 변화가 그래픽스 렌더 스레드 및 상태 변화에 지연 없이 즉각 동기화됩니다.
* **Graphics & Pipeline Isolation (D3DClass):**
  `GraphicsClass`는 렌더링에 필요한 모든 상위 개념을 관리하며, 저수준 Direct3D 하드웨어 제어권은 `D3DClass`로 완전히 격리시켰습니다. Viewport State 변화, BackBuffer SwapChain, Depth-Stencil Buffer 제어 등이 상위 게임 로직과 얽히지 않도록 차단하여 아키텍처의 결합도를 낮췄습니다.
* **Shader & Model Resource Architecture:**
  `ModelClass`는 훈련소 Props 및 묠니르의 버텍스/인덱스 버퍼 데이터 제어를 전담합니다. 렌더링 시점에는 `ColorShaderClass` 및 `TextureShaderClass`와 동적으로 바인딩되어 DeviceContext 상에서 파이프라인 스테이지(IA, VS, RS, PS, OM)를 순차적으로 구동하는 최적의 리소스 파이프라인을 구축했습니다.



[braynzar]: https://www.braynzarsoft.net/viewtutorial/q16390-4-begin-drawing
[Rastertek]: https://www.rastertek.com/dx11win10tut04.html