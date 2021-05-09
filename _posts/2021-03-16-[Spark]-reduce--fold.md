---

title: "[Spark] reduce / fold"
date: 2021-03-16 00:00:00
categories:
- spark
tags: [spark]

---



# Reduce

reduce 함수는 rdd에 있는 임의의 값 2개를 가지고 조합된 결과를 계속 누적하며 마지막으로 최종 결과를 반환하는 함수입니다. reduce는 함수를 인자로 받습니다.

js의 reduce와 동일한 기능을 갖고 있습니다.

단, reduce 연산이 스파크 환경에서 작동하는 과정을 생각해 볼 필요가 있습니다. 스파크는 클러스터 환경에서 동작하는 분산 처리 프로그램입니다.

즉 reduce 연산이 차례대로 첫번째부터 마지막까지 일관되게 실행되는 게 아니라, 파티션별로 나누어 처리된다는 것입니다.

```python
rdd = sc.parallelize([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 2)
```

이런 rdd에 reduce 메서드를 사용한다면,

```python
[1, 2, 3, 4, 5] => reduce 함수 결과   
[6, 7, 8, 9, 10] => reduce 함수 결과
```

n개의 파티션에서 나온 결과를 병합해 최종 결과를 도출해낼 것입니다.

위에서 나눈 `[1, 2, 3, 4, 5] / [6, 7, 8, 9, 10]` 이라는 2개의 파티션은 실제로 이렇게 나누어진다고 장담할 수 없습니다. 물론 이 경우는 간단하기 때문에 실제로도 저렇게 딱 절반으로 나누어질 확률이 높지만, RDD의 파티션이 어떻게 나누어질지는 프로그래머가 정확하게 알기 어렵기 때문에 reduce의 병합 연산도 어떻게 될 지 정확하게 알 수 없습니다.

그렇기 때문에 reduce의 연산은 교환법칙`(a*b = b*a)`이 성립해야 하고, 결합법칙`(a*b)*c = a*(b*c)` 또한 성립해야 합니다. 즉 연산의 순서와 상관없이 같은 결과값이 보장되는 연산이어야 합니다.

# fold

reduce와 비슷한 기능을 하는 fold 메서드입니다. reduce와 차이점은 초기값을 지정할 수 있다는 점입니다.

한 번 fold가 어떻게 작동하는지 보겠습니다.

```python
rdd = sc.parallelize([5, 4, 3, 2], 4)
result_fold = rdd.fold(3, lambda x,y: x+y)
print(result_fold)

# 29
```

defualt value를 3으로 주었더니 [5, 4, 3, 2]의 fold 연산 결과로 29가 나왔습니다. 어떻게 이런 값이 나오는걸까요?

fold 메서드의 생김새를 한 번 보겠습니다.

```python
def fold(self, zeroValue, op):
        op = fail_on_stopiteration(op)

        def func(iterator):
            acc = zeroValue
            for obj in iterator:
                acc = op(acc, obj)
            yield acc
        # collecting result of mapPartitions here ensures that the copy of
        # zeroValue provided to each partition is unique from the one provided
        # to the final reduce call
        vals = self.mapPartitions(func).collect()
        return reduce(op, vals, zeroValue)
```

일단 fold의 내부 함수 func을 보면, 인자로 준 함수`lambda x, y: x+y` 를 가지고 fail_on_stopiteration의 인자에 넣은 뒤 새로운 함수를 반환합니다.

`fail_on_stopiteration`은 특별한 건 없습니다. 함수를 그대로 반환하되, 만약 StopIteration이 발생할 경우(iteration한 RDD에서 만약 next()가 없는데도 도는 경우를 말하는 것 같습니다.) RuntimeError를 발생시키는 wrapper라는 함수를 반환합니다.

```python
def fail_on_stopiteration(f):
    """
    Wraps the input function to fail on 'StopIteration' by raising a 'RuntimeError'
    prevents silent loss of data when 'f' is used in a for loop in Spark code
    """
    def wrapper(*args, **kwargs):
        try:
            return f(*args, **kwargs)
        except StopIteration as exc:
            raise RuntimeError(
                "Caught StopIteration thrown from user's code; failing the task",
                exc
            )

    return wrapper
```

그 다음은 `def func(iterator)`입니다.

```python
def func(iterator):
    acc = zeroValue
    for obj in iterator:
        acc = op(acc, obj)
    yield acc
```

우리가 default value로 주었던 값(zeroValue)을 acc로 두고, acc에 op 연산의 결과를 계속 누적합니다.

그 다음은 `vals = self.mapPartitions(func).collect()` 입니다.

특별할 건 없고, 인자로 전달하는 함수를 파티션 단위로 적용하고 그 결과로 구성된 새로운 RDD를 반환합니다. collect 메서드를 끝으로 누적 연산에 대한 값을 적용한 rdd의 list 형태가 최종 반환될 것입니다.

마지막으로 `return reduce(op, vals, zeroValue)`입니다. python functools 라이브러리의 메서드인 reduce를 가지고 연산을 실행합니다.

즉, 위의 fold의 마지막 형태는

```python
reduce(lambda x, y:x+y, [8, 7, 6, 5], 3)
```

가 될 것입니다.

3 + 8 + 7 + 6 + 5의 결과가 29이기 때문에, 맨 위 fold 연산의 값 또한 29가 나왔습니다.