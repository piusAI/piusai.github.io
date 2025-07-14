---
layout: post
published: true
title: "파이썬 코드를 더욱 강력하게: assert, try-except, yield"
subtitle: "안정성, 효율성의 코드 작성을 위한 핵심 키워드"
cover-img: 
thumbnail-img: 
share-img: 
tags: [python, programming, assert, try-except, yield, generator, exception-handling, code-quality]
author: pius
---

{: .box-note}
**Note:** 이 글은 파이썬 개발자가 코드의 안정성, 효율성, 그리고 가독성을 한 단계 끌어올릴 수 있는 세 가지 핵심 키워드: `assert`, `try-except`, `yield`에 대해 깊이 있게 다룹니다.

---

## 🧷 코드를 더욱 견고하게 만드는 핵심 키워드

파이썬은 배우기 쉽고 강력한 언어이지만, 단순히 문법을 아는 것을 넘어 코드를 더욱 견고하고 효율적으로 만드는 고급 기능들을 이해하는 것이 중요합니다. 이번 글에서는 개발 과정에서 논리적 오류를 빠르게 찾아내고, 프로그램이 예기치 않게 종료되는 것을 방지하며, 대용량 데이터를 메모리 효율적으로 처리할 수 있게 해주는 세 가지 필수 키워드에 대해 알아보겠습니다.

---

### 01 **`assert`: 개발 단계의 안전망**

`assert` 문은 **코드의 특정 시점에서 어떤 조건이 반드시 참(True)이어야 한다고 가정할 때** 사용됩니다. 만약 조건이 거짓(False)이면 `AssertionError`를 발생시켜 프로그램 실행을 즉시 중단시킵니다. 이는 주로 디버깅이나 코드의 유효성 검사를 위해 사용되며, 개발 단계에서 논리적인 오류를 조기에 발견하는 데 매우 유용합니다.

**핵심 역할:** 마치 "만약 이 조건이 참이 아니라면, 이건 심각한 문제이니 바로 멈춰!"라고 선언하는 것과 같습니다. 프로그램을 잘못된 상태로 계속 진행시키는 것을 방지하여, 예상치 못한 버그가 발생하기 전에 문제를 파악할 수 있게 돕습니다.

**예시:**
```python
def calculate_average(numbers):
    assert isinstance(numbers, list), "Input must be a list."
    assert len(numbers) > 0, "List cannot be empty."
    return sum(numbers) / len(numbers)

# 올바른 사용
print(calculate_average([1, 2, 3]))

# 잘못된 사용 (AssertionError 발생)
# print(calculate_average([]))
# print(calculate_average("not_a_list"))
```

---

### 02 `try-except-finally`: 오류 처리 기술
try-except-finally 문은 프로그램 실행 중 발생할 수 있는 예상치 못한 오류(예외, Exception)를 우아하게 처리하여 프로그램이 비정상적으로 종료되는 것을 방지하고, 안정적으로 동작하도록 만듭니다.

핵심 역할: try 블록에서 문제가 생기면 except 블록으로 이동하여 특정 오류에 대한 대체 로직을 실행하는 방어막 역할을 합니다.
finally 블록은 오류 발생 여부와 관계없이 항상 특정 작업을 수행하게 하여, 자원 정리(파일 닫기, 네트워크 연결 해제 등)를 보장합니다.


```python
def read_file_safely(filename):
    try:
        with open(filename, 'r') as f:
            content = f.read()
        print(f"파일 내용:\n{content}")
    except FileNotFoundError:
        print(f"오류: '{filename}' 파일을 찾을 수 없습니다.")
    except Exception as e: # 예상치 못한 다른 모든 예외 처리
        print(f"파일 읽기 중 알 수 없는 오류 발생: {e}")
    finally:
        print("파일 처리 시도 완료.")

# 올바른 파일
read_file_safely("existing_file.txt") # 'existing_file.txt'가 있다고 가정

# 없는 파일
read_file_safely("non_existent_file.txt")
```

---
### 03 `yield (Generator)`: 메모리 효율성의 마법사

yield는 함수를 **제너레이터(Generator)**로 만들어주는 키워드입니다.
일반 함수가 모든 결과를 한 번에 생성하여 메모리에 저장하고 return하는 것과 달리, 제너레이터 함수는 yield 키워드를 통해 값을 하나씩 '생성'하고 '반환'하면서 함수의 실행 상태를 일시 정지시킵니다.
그리고 __next__() 또는 for 루프에 의해 호출될 때마다 중단되었던 지점부터 다시 실행을 재개합니다.

핵심 역할: 이는 대용량 데이터 처리나 무한 시퀀스를 다룰 때 메모리를 매우 효율적으로 사용할 수 있게 해주는 강력한 기능입니다.
모든 데이터를 한 번에 메모리에 올리지 않으므로, 메모리 부족 문제를 피하고 성능을 최적화할 수 있습니다.

```python
import numpy as np

def number_generator():
    n = 0
    while n < 3:  # 0부터 2까지 3번 yield
        yield n
        n += 1
        
gen = number_generator() # 제너레이터 객체 생성

X = np.arange(7)  # numpy 배열 생성: [0 1 2 3 4 5 6]

try:  # 예외 처리 블록 시작
    for i in iter(X): # X의 요소를 순회
        value = gen.__next__() # 제너레이터에서 다음 값 요청
        print(value)
        if value == 7:  # 이 조건은 실제로는 실행되지 않음
            break
            
except StopIteration as e: # 제너레이터가 더 이상 값이 없을 때 발생
    print("nono")

finally: # 예외 발생 여부와 관계없이 항상 실행
    print('done!')
```
---
##🧘‍♂️ 결론: 코드의 품질을 높이는 핵심 도구들
assert, try-except, yield는 파이썬에서 코드를 작성할 때 단순히 기능을 구현하는 것을 넘어, 코드의 품질과 안정성, 그리고 효율성을 크게 향상시키는 중요한 도구들입니다.

- assert로 개발 중 논리적 오류를 조기에 발견하고,

- try-except로 예상치 못한 예외를 우아하게 처리하여 프로그램의 안정성을 높이며,

- yield를 활용한 제너레이터로 대용량 데이터를 메모리 효율적으로 다룰 수 있습니다.

이러한 키워드들을 적재적소에 활용하는 연습을 꾸준히 한다면,
더욱 견고하고 최적화된 파이썬 애플리케이션을 개발하는 데 큰 도움이 될 것입니다.
파이썬의 진정한 힘을 경험해 보세요!



    
