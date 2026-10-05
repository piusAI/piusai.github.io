---
layout: post
published: true
title: 함수 포인터 | Function Pointer
date: 2026-09-30 21:13:00 +0900
description: "-"
thumbnail-img:
categories:
  - C++
tags:
  - cpp
  - ComputerScience
---
함수 포인터는 변수를 포인터로 받는 일반 포인터에서 더 나아가 함수를 주소를 가리키는 것이다.
다시말해, 아래와 같이 함수 **주소**를 저장한 것이다. 
- 함수 자체는 Code 영역의 일련 명령어들이 쭉 이어져있는 Data이다.  
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/cpp/funcptr.png" alt="funcptr" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>함수를 가리키는 포인터</strong> </td>  </tr> </table>

> 포인터 변수 자체는 **stack**에 있고, 그 pointer가 가리키는 대상(함수)는 Code 영역에 있다.
> `main()`이 돌면서 stack frame이 만들어질떄 `ptr`확보

### FuncPtr : 함수 포인터의 이해
#### 타입만 포함:

``` cpp

void print()
{
	cout<<"Thank you FuncPtr! "<<endl;
}
int main()
{
	using FuncPTR = void();
	FuncPTR* ptr = &print; // &:명시적 참조, 안적으면 암묵적 참조로됨
	
	ptr();
}

```

함수 타입만 함수 포인터로 지정하는 방식

#### 포인터까지 타입에 포함:

``` cpp

void print()
{
	cout<<"Thank you AI"<<endl;
}
int main()
{
	using FuncPTR2 = void(*)();
	FuncPTR ptr2 = &print; 
	
	ptr2();
}

```

**과거 방식**:
``` cpp
Typedef void FuncPtrType(); // using FuncPtrType = void();

Typedef void(*FuncPtrType)();  //using FuncPtrType = void(*)();
FuncPtrType ptr = &Print;
```

DirectX에서도 이렇게 활용했었음

``` cpp
// 1. 함수 정의
LRESULT CALLBack WndProc(HWND hwnd, UINT msg, WPARAM wParam, LParam, lParam){} //이 함수를 등록


// 2. 함수 포인터 타입
using WndProcType = LRESULT(CALLBACK*)(HWND, UINT, WPARAM, LPARAM);

// 3. 등록
WNDCLASS wc = {};
wc.lpfnWndProc = &WndProc;; //함수 포인터로 등록 - 명시적 주소연산자
```
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/cpp/lpfnWndProc.png" alt="lpfnWndProc" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>lpfnWndProc - 함수포인터</strong> </td>  </tr> </table>

1. 함수 포인터 선언
2. 함수 포인터 주소 담기
3. 함수 포인터 호출


### FuncPtr : 함수 포인터의 활용 이유

행동 자체를 인자로 넘기고 싶을때 유용하다

역으로 함수를 호출하는 **CallBack**과 같은 패턴을 만들때 쓰는 수단
``` cpp
UEhancedInputComponent::BindAction(JumpAction, ETriggerEvent::Triggered, this, &MyChar::Jump);
// JumpAction 입력 오면 Jump()호출해달라고 등록
// 때가 되면 알아서 호출
```

- Delegate : UE의 안전한 함수 포인터
- BindAction : Delegate를 Input에 연결
- Context Mapping : Context에서 그 Delegate 활성화 할지를 결정
- `&MyChar::Jump`는 일반 함수 포인터가 아니라 **멤버 함수 포인터**이다.
  그렇기 때문에 `BindAction`에 `This`를 같이 넘기는 것이다.

#### CallBack:
``` cpp
void Run(void (*callback)()) //행동을 인자로 받음
{
	...
	callback();     // 때가 되면 호출
}

Run(&print);        //print를 "나중에불러줘"
```
내가 직접 부르는 것이 아니라, 다른 **엔진, Library, Event system**이 나중에 부르도록 넘겨준다.


Skill 부를때
```cpp
void Fire() {}


int main()
{
	using OnClickKeyboard = void(*)();
	OnClickKeyboard Qskill = Fire; //Skill에 대한 실행 함수 자체 저장

	Qskill();

}
```



item선택:
```cpp
using ItemSelectorType = bool(*)(Item* item);
Item* FindItem(Item items[], int itemCount, ItemSelectorType selector)
{
	for (int i = 0; i < itemCount; i++)
	{
		Item* item = &items[i];
		if (selector(item))
			return item;
	}
	return nullptr;
}

bool isRarity(Item* item) { return item->_rarity == 2; }
```

이렇게 Rarity 찾는 함수를 만들 수 있다.
동작도 data화 해서 Function의 parameter로 넘겨주면 이렇게 다양한 상황에서 응용할 수 있다!

##### 멤버 함수 포인터:
함수 호출시, 전달되는 인자 순서/ 스택은 누가 정리할지

|       | 호출 규약    |
| ----- | -------- |
| 일반 함수 | cdecl    |
| 멤버 함수 | thiscall |

```cpp
//일반 함수 -> __cdecl(기본값)
void Print(){}
//사실 위아래 동일
void __cdecl Print(){}

//멤버 함수 -> __thiscall(자동 적용)
Class Item{
	void Print(){}
	//사실 위아래 동일
	void __thiscall Print(){}
}
```
- 안써줘도 알아서 전처리가 해주기는 한다.

``` cpp
class Test{
public:
	void print() { cout << "Its' Member FuncPtr!" << endl; }
};

int main()
{
	using FuncPtrType = void(Test::*)();
	FuncPtrType func = &Test::print;

	Test* t = new Test;
	(t->*func)();
}
```
- 이렇게 멤버 변수 내에서의 함수포인터!

- 서버와 클라에서 사용하기 좋다!
- 주문 넣어주고 할때의 순차적으로 실행

- 함수 포인터는 어떤 동작을 담아둘 수있지만, 데이터 담아둘 수 없다.
- 데이터를 담아두려면 **functor(함수 객체)** 를 이용해야한다!