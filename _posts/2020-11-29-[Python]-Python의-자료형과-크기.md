---
title: "[Python] Python의 자료형과 크기"
date: 2020-11-29 00:00:00
categories:
- python
tags: [python]
---

Python은 자료형이 없다. 없는건 아니고 프로그래머가 직접 자료형을 선언하지 않는다.

내부적으로 C로 구현된 PyObject라는 공통 구조체가 있고, 이를 거치면서 자료형에 맞는 클래스를 부여받는 방식으로 Python의 변수들은 자료형을 가진다.

그 크기는 어떨까?

```python
sys.getsizeof(2)
# >> 28

sys.getsizeof(3.0)
# >> 24
```



> `sys.getsizeof`(*object*[, *default*])
>
> Return the size of an object in bytes. The object can be any type of object. All built-in objects will return correct results, but this does not have to hold true for third-party extensions as it is implementation specific. - https://docs.python.org/3/library/sys.html

<br/>

object의 크기를 byte로 반환해준다고 했는데, 정수와 실수의 크기를 출력하니 28과 24가 나온다.

Python에서 int, str, float 등 자료형은 전부 클래스이기 때문에, 정수 하나 크기의 byte 뿐만 아니라 class의 메타 정보들까지 전부 가져와서 저런 큰 수의 byte를 return하는 것으로 보인다. 객체에 기본적으로 오버헤드도 할당할테니, 그 크기까지 포함된 듯 하다.

<br/>

참고 : 

**PyObject**: https://docs.python.org/ko/3/c-api/structures.html

**getsizeof return large value** : https://stackoverflow.com/questions/10365624/sys-getsizeofint-returns-an-unreasonably-large-value

