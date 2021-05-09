---
title: "[Pandas] DataFrame iterate 7가지 방법 비교"
date: 2021-01-12 00:00:00
categories:
- pandas
tags: [pandas]
---

pandas dataframe을 가공하면서 더 빠른 방법을 찾다보디 총 7가지 방법을 찾았습니다. Jupyeter notebook 환경에서 측정했고 정확한 시간은 환경마다 다를 수 있습니다.



https://www.kaggle.com/residentmario/ramen-ratings

Kaggle에서 2580 row * 7 column을 가진 라멘 평점 (ramen-ratings.csv) 파일을 가져와 Dataframe을 만들었습니다.

각 row의 Review, Stars 컬럼 값에 따라 새로 만드는 컬럼 Feedback의 값을 Good, Not bad, No idea 중 하나로 정하는 연산을 수행했습니다.

Jupyter notebook 환경에서 진행했으며, 구체적인 속도는 실행환경에 따라 다를 수 있습니다.

```python
ramen_df = pd.read_csv('ramen-ratings.csv')
```



<br/>

# 1. for loop

가장 기초적인 방법입니다. Dataframe을 for loop로 돌며 iloc 메서드로 Row를 특정해 컬럼 값을 변경합니다.

```python
def for_loop():
    ramen_df['Feedback'] = ['No idea'] * len(ramen_df)
    for row in range(0, len(ramen_df)):
        review, star = ramen_df['Review #'].iloc[row], ramen_df['Stars'].iloc[row]
        if star != 'Unrated':
            if ((review > 2000) & (float(star) > 3.5)):
                ramen_df['Feedback'].iloc[row] = 'Good'
            elif ((review > 1000) & (float(star) > 3.0)):
                ramen_df['Feedback'].iloc[row] = 'Not bad'

```

```python
%%timeit
for_loop()
```

```python
610 ms ± 17.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```



610ms가 나왔습니다. 걸린 시간에 비해 표준편차도 크게 나왔기 때문에 평균 시간에 대한 신뢰도도 떨어지고, 직관적으로 보더라도 좋은 방법은 아님을 알 수 있습니다.

<br/>

# 2. iterrows()

pandas dataframe에서 제공되는 iterrows() 메서드를 이용해 row를 순회합니다. column / row를 Series 단위로 읽기 때문에, for range로 도는것보다는 빠른 속도가 예상됩니다.

https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.iterrows.html

```python
def iter_loop(rating, stars):
    feedback = ''
    if rating > 2000 & (stars != 'Unrated' and  float(stars) > 3.5):
        result = 'Good'
    elif rating > 1000 & (stars != 'Unrated' and float(stars) > 3.0):
        feedback = 'Not bad'
    else:
        feedback = 'No idea'
    return feedback
```

```py
%%timeit
feedback_series = []
for idx, row in ramen_df.iterrows():
    feedback_series.append(iter_loop(row['Review #'], row['Stars']))
ramen_df['Feedback'] = feedback_series
```

```python
234 ms ± 1.18 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```


for range로 도는것보다는 속도가 꽤 빨라졌습니다. 

<br/>

# 3. itertuples()

row를 tuple 단위로 반환해주는 itertuples 함수입니다. iterrows와 크게 다르지 않습니다.

```python
%%timeit
feedback_series = []
for row in ramen_df.itertuples():
    feedback_series.append(iter_loop(row[1], row[6]))
ramen_df['Feedback'] = feedback_series
```

```python
4.18 ms ± 390 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```

엄청난 속도 차이를 보입니다. 왜일까요?

Jupyter notebook에는 %%prun 매직 함수가 있습니다. 노트북 cell이 작동하면서 작동하는 함수, 개수, 시간 등을 알려줍니다. iterrows와 itertuples의 %%prun을 직접 찍어보고 내부에서 함수가 얼마나 돌았는지 확인해보겠습니다.

<br/>

## iterrows

function call : 709863

time : 0.383sec

![image](https://user-images.githubusercontent.com/52685258/104318205-6f54a780-5522-11eb-8ace-903ed42b3af2.png)

<br/>

## itertuples

function call : 14493

time : 0.004sec

![image](https://user-images.githubusercontent.com/52685258/104318237-78de0f80-5522-11eb-9ee9-8c5dd87b0350.png)

같은 로직을 수행했지만 내부 함수호출 개수와 결과는 엄청난 차이를 보입니다. 이러한 이유로 itertuples는 매우 빠른 속도를 보여줍니다.

<br/>

# 3. apply

dataframe / series의 각 요소에 일괄적으로 함수를 적용시키는 apply 함수입니다. 위에서 사용한 iter_loop 함수를 apply로 적용시켜보겠습니다.

https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.apply.html

```python
%%timeit
ramen_df['Feedback'] = ramen_df.apply(lambda row: iter_loop(row['Review #'], row['Stars']),axis=1)
```
```python
22.5 ms ± 1.21 ms per loop (mean ± std. dev. of 7 runs, 10 loops each
```

apply 함수는 새로운 Series나 Dataframe을 return하기 때문에 DataFrame이 너무 클 경우에는 오버헤드 또한 매우 클 수 있습니다.

그러나 apply 함수는 Cython iterater를 기반으로 돌기 때문에 연산속도가 매우 빠릅니다. 따라서 일반적인 경우에는 apply 함수를 적용하는 것이 더 이득입니다.

<br/>

# 4. get_value, set_value

```python
def get_set_value(df):
    df['Feedback'] = 'No idea'
    for i in df.index:
        star, review = df._get_value(i, 'Stars'), df._get_value(i, 'Review #')
        if star != 'Unrated':
            if review > 2000 and float(star) >= 3.5:
                    df._set_value(i, 'Feedback', 'Good')
            elif review > 1000 and float(star) >= 3.0:
                    df._set_value(i, 'Feedback', 'Good')
    return df
```
```python
%%timeit
result_df = get_set_value(ramen_df)
```
```python
12 ms ± 725 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```

for loop로 row를 순회하면서 get_value / set_value 메서드로 값을 찾아오고, 넣어줍니다.

여기서는 테스트 목적으로 dataframe을 인자로 주고 가공한 dataframe을 반환하는데도 apply보다는 더 빠른 속도를 보여줬습니다. 내부 호출함수도 훨씬 적기 때문에 이 속도보다 더 빠른 속도를 기대할 수 있습니다.

<br/>

# 5. pandas series vectorization

dataframe의 Column을 하나의 series로 보고, loc 함수로 vectorization된 series에 값을 일괄 적용합니다. 함수를 적용시키기가 번거로워 `Stars` 컬럼에 있는 'Unrated'를 전부 -1.0으로 변환시킨 뒤 astype으로 자료형을 float으로 바꿔주는 작업을 선행했습니다.

```python
def vector(review, stars):
    stars[stars == 'Unrated'] = -1.0
    stars = stars.astype(np.float64)
    ramen_df['Feedback'] = 'No idea'
    ramen_df.loc[((review >= 2000) & (stars >= 3.5)), 'Feedback'] = 'Good'
    ramen_df.loc[((review > 1000) & (review < 2000)) | ((stars >= 3.0) & (stars < 3.5)), 'Feedback'] = 'Not bad'

%%timeit
ramen_df['Feedback'] = vector(ramen_df['Review #'], ramen_df['Stars'])
3.79 ms ± 200 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```
```python
%%timeit
ramen_df['Feedback'] = vector(ramen_df['Review #'], ramen_df['Stars'])
```
```python
3.79 ms ± 200 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```
속도 뿐만 아니라 표준편차 또한 크게 감소했습니다. vector의 힘이 새삼 느껴집니다.

<br/>

# 6. numpy narray vectorization

pandas.Series.values는 numpy.ndarray 타입을 갖습니다. ndarray의 벡터 연산은 pandas보다 더 빠를거라 기대합니다. 메서드는 vector를 그대로 사용합니다.

```python
%%timeit
ramen_df['Feedback'] = vector(ramen_df['Review #'].values, ramen_df['Stars'].values)
1.06 ms ± 43.1 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
```
```python
1.06 ms ± 43.1 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
```

맨 처음 시도했던 for loop보다 600배 가량 빠른 속도가 나왔습니다.

<br/>

# 결론

예상한대로 python for loop가 가장 느리고, numpy ndarray vector 연산이 가장 빠릅니다. 둘은 600배 이상의 속도 차이를 보였으며, vector 행렬 연산의 특성상 이 차이는 dataframe의 크기가 더 커질수록 극명해질 것입니다.

그러나 vector 연산은 하나의 조건이 일괄적용되기 때문에 column을 순회하며 다양한 조건이 있는 함수를 적용시키기가 번거롭습니다.햣 

- column의 각 row에 조건을 걸거나 직접 만든 함수를 적용시키는등 예외조건이 많을때는 apply를 추천합니다. get_value / set_value보다 깔끔하게 직관적인 코드 작성이 가능합니다.
- 특별한 예외조건 없이 일괄적으로 함수를 적용할때는 numpy vector가 압도적입니다.