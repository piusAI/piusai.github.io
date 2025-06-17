---
layout: post
title: "2D HumanPose Estimation : A survey"
date: 2025-06-15
categories: ai paper
tags: [paper, pose estimation, English study, ai]
author: pius
published: true
cover-img: /assets/img/POSE.jpg
thumbnail-img: /assets/img/PaperThumnail.png
share-img: /assets/img/POSE2.jpg
---

## 📄 Survey Title : 2D human Pose Estimation : A Survey

 Paper Link: [2D Human Pose Estimation : A Survey (arXiv)](https://arxiv.org/abs/2204.07370)


This post is for clean-up paper about 2d pose estimation
especially Learn like NN, connective from Neuron - Ai, 2D pose estimation Paper

---

### 📖 Abstract

> Human pose estimation aims at localizing human
anatomical keypoints or body parts in the input data (e.g.,
images, videos, or signals). It forms a crucial component in
enabling machines to have an insightful understanding of the
behaviors of humans, and has become a salient problem in
computer vision and related fields...

---

### 🟨 01. Word Estimate - Label

| Word (t)            | ŷ | Loss |
| ---------------------- | ------- | ---- |
| anatomical (해부학적의)     | 해부학적의   | 0.01 |
| crucial (결정적인)         | 중요한     | 0.02 |
| salient (두드러지는, 핵심적인)  | 중대한     | 0.15 |
| representations (표현들)  | 대표적인    | 0.35 |
| significantly (상당히)    | 특히      | 0.42 |
| reap (get, 수확하다)       | 빠르게     | 0.78 |
| briefly (간단히)          | 간략하게    | 0.03 |
| refinement (다듬기, 개선)   | 재정재     | 0.65 |
| robust (튼튼한, 견고한)      | 로봇과같은   | 0.88 |
| recognition (인식)       | 인지      | 0.05 |
| representational (표현의) | 대표적인    | 0.28 |
| incorporates (포함하다)    | 관계되는    | 0.52 |
| agnostic (중립적인)        | 친화적인    | 0.91 |
| polishing (마무리 손질)     | 풀잎      | 0.95 |
| strategy (전략)          | 전술      | 0.04 |
| methodological (방법론적인) | 방식론적인   | 0.01 |
| systematic (체계적인)      | 시스템적인   | 0.05 |


| Word (정답 t)           | ŷ (예측값)  | Loss |
| --------------------- | -------- | ---- |
| prevent (막다, 방지하다)    | 앞의       | 0.85 |
| tackling (씨름하다, 다루다)  | 어려움      | 0.40 |
| Occlusion (폐색, 차단)    | 혼동       | 0.55 |
| entanglement (엉킴, 얽힘) | 자세       | 0.80 |
| absence (부재, 없음)      | (예측값 없음) | 1.00 |
| cope (대처하다, 대응하다)     | 해결하다     | 0.10 |


Low Loss : Methodological, Anatomical
High Loss : Robust, Agnostic, Polishing


---

### 🧠 02. 의미 추론 및 해석 연습

> “Realtime multi-person 2D pose estimation is a key component...”  
→ 영상과 이미지에서 사람을 이해하는 데 중요한 요소는 바로 **실시간으로 여러 사람의 2D 자세를 추정**하는 기술이다.

> “...Part Affinity Fields (PAFs), to learn to associate body parts...”  
→ **PAF라는 비모수적 표현**을 이용해 사람의 **각 신체 부위를 식별하고 연결**한다.

---

### 📘 03. 단어 확인 후 다시 읽기

위에서 정리한 단어들을 확인한 뒤, 전체 abstract를 다시 한 번 천천히 읽어보며 문장의 의미를 정확히 파악합니다.  

---

### ✅ 04. Summary

> 이 논문은 이미지 안의 **여러 사람의 자세를 실시간으로 추정하는 알고리즘**을 소개한다.  
> 핵심은 **Part Affinity Fields(PAF)** 라는 방식을 통해 **신체 부위의 연결성을 학습**하고, 이 연결을 바탕으로 각 사람의 자세를 파악하는 것이다.  
> bottom-up 방식이므로 **사람 수에 상관없이 빠르고 정확한 추정이 가능**하다.

---

### 🧩 Tag Keyword

#PoseEstimation #PAF #2DPose #EnglishPaperStudy #NN방식학습 #논문정리 #piusAI
