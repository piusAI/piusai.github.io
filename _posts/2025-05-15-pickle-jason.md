---
layout: post
title: "Pickle vs JSON, MA vs MB"
published: false
subtitle: "파일도 철학이다"
gh-repo: piusai/ai-devlog
gh-badge: [star, fork, follow]
tags: [python, pickle, json, maya, 파일관리]
comments: true
mathjax: false
author: Pius
---

{: .box-note}
**Note:** 이 글은 파이썬의 파일 저장 방식과 Maya의 파일 포맷 차이에 대해 정리한 글입니다.  
AI와 CG를 병행하며 마주친 아주 실질적인 선택의 갈림길.

---

## 🧠 Pickle vs JSON

### 📌 JSON

- 사람이 읽을 수 있는 **텍스트 기반 포맷**
- `{ key: value }` 구조
- 대부분의 언어/툴과 호환
- **클래스 인스턴스 저장 불가**

```json
{
  "name": "pius",
  "age": 29
}
```

### 📦 Pickle

- 파이썬 전용 **바이너리 포맷**
- 클래스 인스턴스, 넘파이 배열, 딥러닝 모델까지 저장 가능
- 보안상 조심해야 함 (임의 파일 로드 시 위험)

```python
import pickle

with open('model.pkl', 'wb') as f:
    pickle.dump(my_model, f)
```

{: .box-warning}
**Warning:** Pickle은 강력하지만, 다른 언어에서는 거의 사용할 수 없습니다.

---

## 🎨 MA vs MB — Maya 파일 포맷

### 🔤 .ma (Maya ASCII)

- 사람이 읽을 수 있는 **텍스트 기반**
- Git으로 버전 관리가 용이
- 무겁고 로딩 속도가 느림
- MEL 스크립트 형식으로 작성되어 있음

```mel
createNode transform -n "Cube1";
```

### 🧱 .mb (Maya Binary)

- **바이너리 형식**
- 용량이 작고 빠르게 로딩됨
- 내용 확인이 불가능하며, 충돌 해결이 어렵다

---

## 📂 Pickle vs JSON / MA vs MB의 공통점

| 비교 항목       | 텍스트 기반 (.json, .ma)              | 바이너리 기반 (.pkl, .mb)         |
|----------------|--------------------------------------|----------------------------------|
| 가독성         | ✅ 좋음                              | ❌ 없음                          |
| Git 충돌 관리   | ✅ 가능                              | ❌ 불가                          |
| 처리 속도      | ❌ 느림                              | ✅ 빠름                          |
| 호환성         | ✅ 높음                              | ❌ 낮음                          |
| 수정/검토 용이 | ✅ 높음                              | ❌ 어려움                        |

{: .box-success}
**결론:**  
사람이 직접 관리하길 원하면 텍스트.  
성능과 용량이 중요하면 바이너리.

---

## 🎯 Devlog Summary

- AI 프로젝트를 하며 `.pkl`을, CG 작업을 하며 `.ma/.mb`를 자주 다루게 됨
- 결국 모든 파일 포맷은 **철학의 문제**
- 목적에 따라 포맷을 선택할 수 있어야 한다

> Just play for data.  
> Just play for life.  
> It's Pius.
