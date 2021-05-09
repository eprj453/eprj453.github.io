---
title: "[Spark] map, flatMap"
Date: 2021-02-25 00:00:00
categories:
- spark
tags: [spark]
---

map과 flatMap은 spark transformation의 대표적인 연산입니다. 이 둘을 사용해보고 차이점이 무엇인지 살펴보겠습니다. pyspark을 이용합니다.

<br/>

# map

spark의 map은 scala나 python에서 제공하는 map과 크게 다르지 않습니다. python에서 제공하는 map은 다음과 같습니다.

1. 함수를 인자로 갖고, 
2. 리스트와 같은 iterable 자료구조의 모든 요소에 그 함수를 적용시키고,
3. 그 결과로 구성된 map 객체를 다시 돌려줍니다.

spark의 map도 자료구조가 RDD라는 점이 다르고 작동은 비슷합니다.

```python
from pyspark import SparkContext, SparkConf

conf = SparkConf()
sc = SparkContext(master="local[*]", appName="hello", conf=conf)

rdd = sc.parallelize([
    'a', 'b', 'c', 'd'
])

rdd2 = rdd.map(lambda x: x+' with map')
print(rdd2.collect())

'''
['a with map', 'b with map', 'c with map', 'd with map']
'''
```

RDD의 모든 요소에 동일한 lambda 함수가 적용된 것을 볼 수 있습니다.



# flatMap

map과 유사하게 동작하지만, flatMap의 반환 타입은 map과는 조금 다릅니다.

아래는 rdd.py 클래스에 정의되어 있는 map과 flatMap입니다.

```python
def map(self, f, preservesPartitioning=False):
    """
        Return a new RDD by applying a function to each element of this RDD.

        >>> rdd = sc.parallelize(["b", "a", "c"])
        >>> sorted(rdd.map(lambda x: (x, 1)).collect())
        [('a', 1), ('b', 1), ('c', 1)]
        """
    def func(_, iterator):
        return map(fail_on_stopiteration(f), iterator)
    return self.mapPartitionsWithIndex(func, preservesPartitioning)

def flatMap(self, f, preservesPartitioning=False):
    """
        Return a new RDD by first applying a function to all elements of this
        RDD, and then flattening the results.

        >>> rdd = sc.parallelize([2, 3, 4])
        >>> sorted(rdd.flatMap(lambda x: range(1, x)).collect())
        [1, 1, 1, 2, 2, 3]
        >>> sorted(rdd.flatMap(lambda x: [(x, x), (x, x)]).collect())
        [(2, 2), (2, 2), (3, 3), (3, 3), (4, 4), (4, 4)]
        """
    def func(s, iterator):
        return chain.from_iterable(map(fail_on_stopiteration(f), iterator))
    return self.mapPartitionsWithIndex(func, preservesPartitioning)
```

map은 python이 제공하는 map만을 사용하는데, flatMap은 itertools의 chain.from_iterable을 사용합니다. 

https://docs.python.org/3/library/itertools.html#itertools.chain.from_iterable

공식문서의 설명을 보아, iterable 객체의 각 요소를 한 단계 더 작은 단위로 쪼개는 기능임을 알 수 있습니다.

```python
[[1, 2, 3], [4, 5, 6]] --> [1, 2, 3, 4, 5, 6]
['abc', 'def'] ==> ['a', 'b', 'c', 'd', 'e', 'f']
```



flatMap의 기능도 쉽게 예측할 수 있습니다. map과 어떻게 다른지 비교해보겠습니다.

```python
rdd = sc.parallelize([
    'a,b,c',
    'd,e,f',
    'g,h,i'
])

map_rdd = rdd.map(lambda x: x.split(','))
print(map_rdd.collect())

flatmap_rdd = rdd.flatMap(lambda x: x.split(','))
print(flatmap_rdd.collect())

'''
[['a', 'b', 'c'], ['d', 'e', 'f'], ['g', 'h', 'i']]
['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i']
'''
```

map의 기능은 rdd 각 요소에 함수를 적용시키는 것까지입니다.

flatMap은 map으로 얻은 결과의 각 요소를 끄집어내는 것까지입니다.



리스트가 아니라 string은 어떨까요?

```python
rdd = sc.parallelize([
    ['a', 'b', 'c'],
    ['d', 'e', 'f']
])

flatmap_rdd = rdd.flatMap(lambda x: ','.join(x))
print(flatmap_rdd.collect())

'''
['a', ',', 'b', ',', 'c', 'd', ',', 'e', ',', 'f']
'''
```

새로운 rdd의 요소가 join으로 얻은 문자열을 전부 펼친 문자인 것을 볼 수 있습니다.

