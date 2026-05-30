---
layout: post
published: false
title:  "RenderPipeline?"
date:   2026-05-29 15:32:00 +0900
description: C++의 OOP를 제거하고 오직 그래픽스 렌더파이프라인에만 집중해본 연구 기록입니다.
img: pius-icon.png                  # 목록에 출력될 썸네일 이미지 (필요시 사용)
categories: [R&D]
tags: [DX11, Graphics, RenderPipeline]
author: PIUS
---

<!-- 1. 포스트 최상단에 렌더파이프라인 메인 이미지 배치 -->
![Render Pipeline Image]({{ '/assets/img/Renderpipeline.jpg' | relative_url }})

---

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight cpp %}
#include <iostream>
using namespace std;

int main() 
{
    cout << "Test Home" << endl; 
    return 0;
}
{% endhighlight %}

---

### 📌 학습 및 구현 참조 레퍼런스

* [Braynzar Soft DX11][braynzar] : C++의 OOP 구조를 걷어내고, 오직 렌더파이프라인 자체에만 집중해서 로직을 구현해 볼 수 있었습니다.
* [Rastertek DX11][Rastertek] : 전체적인 그래픽스 엔진 프레임워크를 설계하고 빌드하면서 깊이 있게 공부해볼 수 있습니다.

[braynzar]: https://www.braynzarsoft.net/viewtutorial/q16390-4-begin-drawing
[Rastertek]: https://www.rastertek.com/dx11win10tut04.html