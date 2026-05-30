---
layout: post
title: "DirectX 11 Framework Architecture 설계"
date: 2026-05-29 15:45:00 +0900
description: 상용 엔진(Unreal)의 API 뒤에 숨겨진 렌더 파이프라인을 제어하기 위한 로우레벨 DirectX 엔진 프레임워크 설계 기록입니다.
categories: [R&D]
tags: [DirectX, Framework, Architecture, Graphics-Pipeline]
---

대다수의 상용 엔진은 그래픽스 파이프라인을 고도로 추상화하여 제공합니다. 하지만 정밀한 전투 컴포넌트의 성능 병목을 해결하고, 대규모 오브젝트의 타격 이펙트를 프레임 저하 없이 렌더링하기 위해서는 **CPU와 GPU 간의 데이터 흐름(Data Flow)**을 완벽하게 통제할 수 있어야 한다 생각합니다.

본 포스팅에서는 상용 엔진 없이 오직 C++와 로우레벨 그래픽스 API만을 활용해 구축한 **DirectX 게임 엔진 프레임워크의 구조적 설계**와 핵심 컴포넌트 간의 결합도 최적화 과정을 공유합니다.

---

## 🧩 1. 프레임워크 핵심 아키텍처 다이어그램

엔진 프레임워크의 계층 구조는 다음과 같이 **[Application 인프라 ➡️ Core 엔진 ➡️ Graphics 파이프라인]**으로 철저히 관심사를 분리(Separation of Concerns)하여 컴포넌트 간의 결합도를 최적화했습니다.



* **WinApp (Entry Point):** OS 윈도우 생성, 메시지 루프 핸들링, 기본 입력 이벤트 수집
* **Engine / GameLoop:** 가변/고정 타임스텝 기반의 초정밀 델타 타임(Delta Time) 제어 및 전체 시스템 동기화
* **Graphics / Device:** 하드웨어 가속기(GPU) 및 내부 파이프라인 상태 객체(PSO) 캡슐화

---

## 💻 2. 핵심 루프 및 Win32/DirectX 가동화 로직

프레임워크의 심장인 핵심 게임 루프와 각 모듈의 초기화 라이프사이클을 담당하는 주요 구현체입니다. OOP 구조를 무분별하게 남용하기보다, 절차적인 데이터 흐름을 안전하게 보장하는 방식으로 설계했습니다.



```
{% highlight cpp %}
// Engine.h - 핵심 프레임워크 컨트롤러 인터페이스
#pragma once
#include <windows.h>
#include <memory>

class Graphics;
class Timer;

class Engine 
{
public:
    Engine();
    ~Engine();

    bool Initialize(HINSTANCE hInst, HWND hWnd, int Width, int Height);
    void Run();

private:
    void Update(float DeltaTime);
    void Render();

private:
    bool                     bIsRunning;
    std::unique_ptr<Graphics> GraphicsPipeline;
    std::unique_ptr<Timer>    EngineTimer;
};
{% endhighlight %}

---