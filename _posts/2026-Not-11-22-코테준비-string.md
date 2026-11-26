---
layout: post
published: false
title: 트리의 전반적인 흐름(자료구조)
date: 2026-11-09 23:10:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
코테 준비중에 STL는 알고있더라도,어떤 Library나 관용구가 손에 안익어서 로직만 그렇게 하면 될 것 같은데? 정도로 하고있다.

여러 패턴을 알아보도록 하자.

코딩 테스트에 있어서 지금 현재 부족한 부분
1. Tool의 관용어구 들이 손에 익지 않아서
   ex) map등 자료구조많이 활용해보지 않음
2. 문제 해결 아이디어가 안떠오를떄, 

### 0. 문자 아스키

``` cpp
char c = 'a';
int code = c;      // 97 (그냥 int로 받으면 아스키 값)
char d = 97;       // 'a'
int idx = c -'a';  // 0 (index 변환)
char e = 'a' + 3;  // 'd'
```

문자 -> 문자열

`const char* cat` 이 것으로 저장해둔다

숫자 ↔ 문자열

순열 / 조합

```cpp
sort(v.begin(). v.end()); //next_permutation은 정렬된 상태에서 시작해야 모든 경우가 나옴
do
{
	//v가 현재 순열
} while(next_permutation(v.begin(), v.end()));
```


#### string 자주 쓰는 것들
```cpp
s.size(); s.length();
s.substr(2,3); //2번 index부터 3글자
s.find("ab");  // 없으면 string ::npos
reverse(s.begin(), s.end());
sort(s.begin(), s.end());

s+= "abc" ; s.push_back('x'); s.pop_back();
transform(s.begin(), s.end(), s.begin(), ::toupper); // 대문자 변환
```


```cpp
int n = stoi("123"); //문자열 → 정수
string s = to_string(123); // 정수 → 문자열
int digit = '7'-'0';  // 문자 숫자 → 정수(7)
```

문자와 숫자가 매핑되어있다  이정도만 알면됨


#### substr

**`substr`의 두 번째 값은 "끝 위치"가 아니라 "자를 글자 수"** .

```cpp
s.substr(시작 인덱스, 글자 수)
s.substr(시작 인덱스)          // 글자 수 생략 → 끝까지
```

`s = "abcdef"`라면:

```cpp
s.substr(0, 3)   // "abc"   0번부터 3글자
s.substr(2, 3)   // "cde"   2번부터 3글자 (2~3번이 아님!)
s.substr(2)      // "cdef"  2번부터 끝까지
```