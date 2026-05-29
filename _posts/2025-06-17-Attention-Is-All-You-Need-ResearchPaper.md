---
layout: post
title: "Attention is All You Need : Transformer Research Paper"
date: 2025-06-17
categories: ai paper
tags: [paper, Transformer, English study, ai]
author: pius
published: false
cover-img: /assets/img/Transformer
thumbnail-img: /assets/img/PaperThumnail.png
share-img: /assets/img/Transformer
---

## 📄 Survey Title : Attention is All You Need
 Paper Link: [Attention is All You Need (arXiv)](https://arxiv.org/pdf/1706.03762)


Attention is All You need is Landmark Resarch Paper in ML!

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

| Word | ŷ | t | loss|
|------|------|-----|---|
| Dominant | 의지하는
| Transduction | 중요한 | 결정적인 | 0.02
| dispending | 중대한 | 두드러지는, 핵심적인 | 0.15
| recurrence | 대표적인 | 표현들 | 0.35
| Paralleizable | 특히| 상당히 | 0.42
| ensemble | 빠르게 |  get, 수확하다| 0.78
| fraction | 간략하게, | 간단히 | 0.03
| parsing | 재정재 | 다듬기, 개선 | 0.65


CEE : 0.32

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
