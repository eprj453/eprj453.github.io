---
title: "[Spark] repartition, coalesce 파헤치기"
date: 2021-01-18 00:00:00
categories:
- spark
tags: [spark, data-enginnering]
---

저는 업무에서 AWS Glue를 사용해 Spark Job을 돌리고 있습니다. 마지막에 파티션을 병합해야 결과 파일이 하나로 모이기 때문에, 파티션 1개로 병합하는 작업을 거칩니다. 이 때 사용할 수 있는 메서드가 repartition, coalesce 입니다.

간단하게 말하면 파티션을 병합하는 과정에서 repartition은 셔플을 실행하고, coalesce는 셔플을 실행하지 않습니다. 이를 비교해놓은 포스팅이나 stackoverflow 글은 상당히 많기 때문에 차이점을 깊게 언급하지는 않겠습니다.

파티션을 줄이는 과정에서는 coalesce, 파티션을 늘려야 하는 경우는 repartition을 쓴다고 알고 있었고 틀렸다고도 생각하지는 않습니다. 

그렇지만 Spark Document에서도 예외를 두고 있는 경우가 있습니다.

> However, if you’re doing a drastic coalesce, e.g. to numPartitions = 1, this may result in your computation taking place on fewer nodes than you like (e.g. one node in the case of numPartitions = 1). To avoid this, you can call repartition(). This will add a shuffle step, but means the current upstream partitions will be executed in parallel (per whatever the current partitioning is).

이번에 마주했던 문제상황도 numPartitions를 1로, 즉 파티션을 하나로 만드는 과정에서 발생했습니다. 처리하는 데이터의 양이 많아지면서 Job 실행시간이 실패 기준으로 정했던 시간을 넘어가 버린 것입니다.

이 문제는 coalesce 메서드를 repartition 메서드로 변경하고 해결되었으나, shuffle을 하지 않음에도 더 많은 시간이 걸린다는 것이 이해가 잘 가지 않아 리니지와 실행 계획들을 살펴보며 공부했습니다.

그 내용을 기록합니다 :)

<br/>

# 문제 상황

spark에서 group by나 join과 같은 wide transformation이 일어나는 경우, 파티션 값을 따로 설정하지 않으면 파티션 갯수가 200개로 늘어납니다. 이대로 RDD나 Dataframe을 바로 파일로 떨구면 파티션의 갯수와 동일하게 여러 파일로 나눠서 저장됩니다.

이를 막기 위해 파티션을 1로 만들어야 했고, coalesce 메서드를 사용했습니다. repartition으로도 가능하지만 repartition은 전체 파티션 셔플이 들어가기 때문에 셔플이 없는 coalesce이 더 나을거라 생각했습니다.

그런데 실제로는 repartition이 훨씬 빠른 속도를 보였습니다. coalesce는 너무 시간이 오래 걸려 끝을 보지 못하고 job을 취소했습니다. 이를 이해하기 위해서는 shuffle이 정확히 어떻게 동작하는지 자세히 볼 필요가 있습니다. repartition과 coalesce의 차이는 shuffle을 하느냐 하지 않느냐, 그것뿐입니다.

<br/>

# spark partition shuffle

















