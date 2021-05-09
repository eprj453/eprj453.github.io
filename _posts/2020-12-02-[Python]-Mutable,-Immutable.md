---
title: "[Python] Mutable / Immutable"
date: 2020-11-29 00:00:00
categories:
- python
tags: [python]
---



mutable 객체는 생성 후에 값을 바꿀 수 없고, Immutable 객체는 생성 후에도 값이 변할 수 있는 객체이다.

메모리 주소를 직접 찍어보며 각 객체의 값이 변할 수 없는지 확인해보자.

```python
i = ''
print("처음 id : {}".format(id(i)))
i += "hello"
print("나중 id : {}".format(id(i)))

# >> 처음 id : 2699478716848
# >> 나중 id : 2699508647920
```

immutable 객체인 str의 값을 변경해보니, id 주소가 아예 바뀌었다. 

맨 처음 선언된 i와 hello을 더한 i는 메모리 상에서 아예 다른 곳에 위치한, 다른 변수임을 알아두자.

<br/>

```python
i = []
print("처음 id : {}".format(id(i)))
i += ['hello']
i.append('world')
print("나중 id : {}".format(id(i)))
print(i)

# >> 처음 id : 1396228379016
# >> 나중 id : 1396228379016
# >> ['hello', ' world']
```

i라는 list에 값을 +=로 더하고 append로 추가를 해도 id값이 변하지 않음을 볼 수 있다. 값을 변경하거나 더해도 같은 변수이다.

<br/>

immutable 객체의 값을 변경하는 것은, **값을 변경하는 것이 아니라 다른 위치에 메모리를 재할당하는 것**이다.