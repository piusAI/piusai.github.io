---
layout: post
title: "KCF Tracker: Fast Object Tracking with FFT"
date: 2025-06-16
categories: ai vision
tags: [object tracking, computer vision, ai, fft, kernel]
author: pius
published: false
cover-img: /assets/img/POSE.jpg
thumbnail-img: /assets/img/PaperThumnail.png
share-img: /assets/img/POSE2.jpg
---


# KCF (Kernelized Correlation Filter) 알고리즘 첫걸음

## 📌 목표
이 문서는 KCF 알고리즘의 기초 개념과 핵심 수학 원리를 정리하고, 향후 구현 및 실습을 위한 기반을 제공합니다.

---

## 🧠 1. KCF란?

KCF (Kernelized Correlation Filter)는 **영상 속 객체를 빠르게 추적하는 알고리즘**으로,  
다음 세 가지 기술이 핵심입니다:

- 📐 **Correlation Filter**: 대상과 주변 영역의 유사도 비교
- ⚙️ **FFT (Fast Fourier Transform)**: 연산을 빠르게 수행
- 🧩 **Kernel Trick**: 비선형 데이터를 정교하게 처리

---

## 🔍 2. 왜 빠른가?

추적 알고리즘의 병목은 **패치 간 유사도 계산**입니다.  
KCF는 이 연산을 시공간에서 하지 않고, **주파수 영역으로 바꿔서** 합니다.

### 💡 핵심 수식
시간 영역의 convolution:
