---
layout: post
title:  "DirectX 11 기반 실시간 궤적 제어 기술 연구: 토르 망치 연습소(Thor's Hammer Training Ground)"
date:   2026-05-29 15:32:00 +0900
description: C++의 OOP 구조를 탈피하여 고성능 그래픽스 렌더파이프라인 및 수리적 실시간 오브젝트 제어에 집중한 R&D 기획 및 연구 기록입니다.
img: Thor_Thumnail.png                  # 목록에 출력될 썸네일 이미지 (필요시 사용)
categories: [R&D]
tags: [RenderPipeline, Graphics, DX11, TrajectoryMath]
author: PIUS
---

<!-- 포스트 최상단 메인 아젠다 이미지 배치 -->
![Thor Architecture Agenda]({{ '/assets/postimg/ThorArchitecture/DXGraphics02.jpg' | relative_url }})

---

본 연구는 생성형 AI 시대에 단순 기능 구현 위주의 포트폴리오에서 벗어나, 그래픽스 저수준 제어 능력을 증명하기 위해 진행된 **DirectX 11 기반 실시간 오브젝트 비행 궤적 및 시스템 아키텍처 구현 프로젝트**입니다. 

C++의 무거운 상속 구조(OOP)를 의도적으로 배제하고 고성능 그래픽스 데이터 흐름과 실시간 수학 연산(Lerp, Bézier 등)에 집중하여 AAA급 스튜디오에서 요구하는 대체 불가능한 R&D 엔지니어링 역량을 확보하는 데 목적이 있습니다.

---

## 01. 코어 컨셉 (Core Concept)

### 1) 아스가르드 가상 훈련소 및 묠니르 비행 프로세스
![Thor Hammer Training Ground Core]({{ '/assets/postimg/ThorArchitecture/DXGraphics03.jpg' | relative_url }})

* **프로젝트 배경:** 토르의 월드인 아스가르드의 가상 훈련소를 배경으로 설정하여, 신화적 무기인 '묠니르(Mjolnir)'의 고유 비행 궤적 프로세스를 DirectX 11 환경에서 실시간 시뮬레이션 데모로 구현합니다.
* **아키텍처 지향점:** 렌더러와 시뮬레이션 로직 간의 커플링을 최소화하여 프레임 레이트 저하 없이 부드러운 카메라 네비게이션과 수학적 궤적 계산을 보장합니다.

### 2) 뷰포트 시점(Top-View / TPS)에 따른 네비게이션 시스템
![Hammer Navigation View]({{ '/assets/postimg/ThorArchitecture/DXGraphics04.jpg' | relative_url }})

* **Top View 마우스 입력 시스템:** 사용자가 탑뷰(Top-View) 시점에서 마우스 드래그를 통해 망치의 이동 위치 및 비행 경로를 자유롭게 드로잉(Draw Path)할 수 있도록 설계했습니다.
* **TPS 카메라 체이싱 알고리즘:** 비행 테스트가 시작되면 입구에서부터 TPS(3인칭 복합 시점) 카메라가 망치의 실시간 이동 궤적을 부드럽게 추적(Chasing)하며 아스가르드 훈련소 내부를 역동적으로 연출합니다.

---

## 02. 배경 프롭 및 환경 구성 (Environment Props)

![Background Props Analysis]({{ '/assets/postimg/ThorArchitecture/DXGraphics05.jpg' | relative_url }})

* **Skybox 및 지형 렌더링:** 광활한 아스가르드의 밤하늘(은하수 패노라마) 이미지를 텍스처 큐브 맵으로 바인딩하여 고품질의 실시간 배경 스카이를 구축합니다.
* **환경 프롭 파이프라인:** 훈련소 내부를 구성하는 아스가르드식 깃발, 기둥, 포탈 게이트 등 핵심 기믹 프롭들을 독립적인 메시 인스턴스로 관리하여 그래픽스 리소스 메모리 적재량을 최적화합니다.

---

## 03. 시스템 아키텍처 오버뷰 (System Architecture Overview)

실시간 연산 최적화와 안정적인 트러블슈팅을 위해, 전체 그래픽스 데모의 핵심 모듈을 다음과 같은 논리 구조로 이원화하여 설계했습니다.

![System Architecture Specification]({{ '/assets/postimg/ThorArchitecture/DXGraphics06.jpg' | relative_url }})

