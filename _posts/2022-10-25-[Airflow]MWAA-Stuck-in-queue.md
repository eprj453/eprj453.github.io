---
title: "[Airflow] MWAA Stuck in queue"
date: 2022-10-25 00:00:00
categories:
- airflow
tags: [airflow, aws]
---



현업에서 온프레미스 Airflow를 거쳐 현재는 AWS Managed Airflow인 MWAA에서 Airflow를 운영하고 있습니다. 다소 가격이 비싸지만 데이터 엔지니어링에 많은 여력을 쏟을 수 없는 경우에는 좋은 대안이 될 수 있습니다.

그러나 알아서 구성해준다는 말은 반대로 말하면 구성해주는 것 외에는 제약이 많이 따른다는 뜻입니다. 좀 안좋게 말하면 해주는거 말고는 아무것도 쓰지 말라는 이야기입니다. 일례로 MWAA의 로깅은 무조건 AWS Cloudwatch를 사용해야 합니다. cloudwatch와 통신 딜레이 때문에 여러 줄의 로그를 한번에 찍으면 task 스케줄링 속도가 느려질 정도로 영향이 꽤 큰 편입니다.

이 정도로 그친다면 충분히 감내할만한 정도이나 최근 MWAA를 사용하며 꽤 큰 이슈 상황을 만났고 완벽하진 않지만 어느정도 이를 해결해 현재는 큰 무리없이 Airflow가 운영상에 동작하고 있습니다.

<br/>

# 문제 상황

worker 리소스가 충분함에도 task가 급격히 몰리는 시간에 일부 task들이 queued 상태에서 멈춰 있는 현상이 발생했습니다. 또한 task가 아무런 exception 로그도 남기지 않고 갑자기 죽어버리는 현상도 함께 발생했습니다.

- min-worker : 10
- max-worker : 20
- 동시 실행 task : 약 100개

mw1.medium의 worker 1개는 최대 10개의 task를 동시 실행할 수 있기 때문에 max-worker가 20개라면 100개 정도의 task는 무난하게 처리할 수 있어야 합니다. 그러나 리소스가 부족하다고 추정할 수 있는 에러 메세지를 띄운 것도 아니고 task를 clear해도 queued 현상은 동일하게 발생했습니다. 그런데 다른 dag에서 분 단위로 실행되는 task는 stuck 현상 없이 잘 실행되는 것이 너무 이상했습니다.

물론 갑자기 task나 dag 숫자가 확 늘어난 것도 아니고 갑자기 리소스를 줄인 것도 아니었습니다. 메모리 문제인가 싶어 max-worker 숫자를 최대치까지 늘려 오토스케일링으로 해결해보려 해도 해결되지 않았습니다. 

결론부터 말하자면 이는 오토스케일링의 불안정성 때문에 일어난 문제입니다. 오토스케일링을 사용하지 않음으로써 이 문제를 해결할 수 있었습니다.

<br/>

# 원인 추론

생각할 수 있는 문제의 원인은

- MWAA 내부의 scheduler <-> Worker 인스턴스 간의 네트워크 문제
  - worker가 오토 스케일링 중에 task를 가지고 있는 채로 죽어버리는게 아닐까?
- MWAA 내부의 메타 데이터 손실
  - 해당 execution_date의 task를 실행할 수 없는 어떤 상태로 인지하고 있다?

정도였습니다.



여기서 managed 서비스의 한계를 느꼈습니다. 메타 데이터의 문제라면 DB에 sql을 날려서 확인해보면 될 문제이고, scheduler와 worker 간의 네트워크 문제라면 서버에 접속해 ping이라도 한번 날려보고 싶은데 그게 불가능하기 때문입니다. 



<br/>

# 원인 발견

최소 10개, 최대 20개의 worker 인스턴스를 오토스케일링으로 사용하고 있었으나, task가 한번 몰리고 나서 다시 다운스케일링이 되지 않는 현상을 발견했습니다. 

<img width="785" alt="image" src="https://user-images.githubusercontent.com/52685258/197803063-c6143649-0de9-48e7-9d3b-a5470c7c8003.png">

특정 시간대를 제외하고는 많아도 4~5개의 worker면 충분히 처리가 가능한 수준인데, 14개의 worker가 몇시간동안 떠있으면서 불필요한 병렬처리가 계속되고 있었습니다. 이렇게 많은 worker가 4~5분에 1개의 task를 처리하고 있다면 이는 낭비입니다. 

명백하게 오토스케일링이 제대로 작동하지 않았고 airflow의 parallelism 옵션을 조정해도 해결되지 않았습니다.

원인은 metric에서 찾을 수 있었습니다.

<img width="2654" alt="image" src="https://user-images.githubusercontent.com/52685258/197806376-aa5c2fda-2427-4ea2-9afc-ca3d5e12ac94.png">



이상한 점은 2가지였습니다.

- 실제 RunningTasks보다 더 많은 Task가 찍히고 있습니다.

  - UTC 21시에 약 200개, 그 외의 시간대에는 30~50개의 Task가 찍히는 것이 맞는데, 비정상적으로 많은 task가 RunningTask 메트릭으로 집계되는 것을 볼 수 있습니다. 이 수치를 기반으로 mwaa의 오토스케일링이 작동하기 때문에 worker 또한 죽지 않고 계속 떠있었다고 볼 수 있습니다.

    > Amazon MWAA uses `RunningTasks` and `QueuedTasks` [metrics](https://docs.aws.amazon.com/mwaa/latest/userguide/access-metrics-cw-202.html#available-metrics-cw-v202), where *(tasks running + tasks queued) / ([tasks per worker](https://docs.aws.amazon.com/mwaa/latest/userguide/environment-class.html#environment-class-sizes)) = (required workers)*. If the required number of workers is greater than the current number of workers, Amazon MWAA will add Fargate worker containers to that value, up to the maximum value specified by `max-workers`. 
    >
    > When the `RunningTasks` and `QueuedTasks` metrics sum to zero for a period of two minutes, Amazon MWAA requests Fargate to set the number of workers to the environment's `min-workers` value. Amazon MWAA provides Fargate a `stopTimeout` value of 120 seconds, currently the maximum available time, to allow any work to complete on the workers, after which the container is removed and any remaining work in progress is deleted. In most cases, this occurs while no tasks are in the queue, however under certain conditions mentioned in [preceding section](https://docs.aws.amazon.com/mwaa/latest/userguide/mwaa-autoscaling.html) of this page, tasks might be queued while downscaling is taking place. - [MWAA Official Doc](https://docs.aws.amazon.com/mwaa/latest/userguide/mwaa-autoscaling.html)

- 그래프가 너무 불안정합니다. 저는 수치가 뻥튀기되는것보다 더 좋지 않은 현상이라고 보았습니다. 인스턴스 상태가 불안정하다면 그 안에서 process 단위로 동작하는 task들의 안정성도 보장할 수 없기 때문입니다.



<br/>

# 문의

AWS Support center에 문의글을 남겼고 인도인 엔지니어에게 질답을 반복하고 화상회의까지 한 결과, 결론은 **오토스케일링 쓰지 말고 min-worker == max-worker로 두고 써라** 입니다.

> So this issue could be due to scale-in at an inconvenient time due to Autoscaling[1], so the workaround is to set min-workers = max-workers to disable autoscaling

음... MWAA의 오토 스케일링 기능이 불안정하게 작동하니 오토 스케일링 자체를 쓰지 마라는 말이 처음에는 좀 화가 났습니다. 공짜로 쓰는 것도 아닌데 managed 서비스를 이런 식으로 가이드해주다니요.

그래도 속는쳄 치고 그렇게 올려보니 확실히 metric이 안정되었습니다. 수치는 여전히 실제보다 뻥튀기되어 있으나 비교적 실제와 비슷해졌고 그래프가 갑자기 널뛰기하는 현상도 거의 사라졌습니다. 물론 stuck in queued 현상도 사라졌습니다.

worker가 오토스케일링 되는 과정에서 AWS 내부 인프라가 제대로 동작하지 않는 것이 합리적인 추측이겠습니다.

<img width="2650" alt="image" src="https://user-images.githubusercontent.com/52685258/197809702-d094cffa-d5f5-423e-8d03-bd00cc6ca134.png">



<br/>

# 그렇다면 어떻게?

문제를 직접 해결하거나 deep dive하기는 어렵습니다. manged 서비스를 사용하는 이상 min-worker == max-worker로 두어 오토스케일링을 하지 않는 이 방법이 최선이라 생각했습니다.

그렇다고 비싼 worker를 한달 내내 최대치 띄워놓고 사용할수도 없는 노릇이니, MWAA의 스케일을 관리하는 DAG을 추가로 개발할 예정입니다.

task가 몰리는 특정 시간동안만 리소스를 고정적으로 크게 할당하고 나머지 시간에는 최소 리소스를 사용하는 식으로 스케줄링을 걸어두면 비용을 줄이면서 어느정도 안정적인 task 운영이 가능해집니다.

물론 좋은 방법은 아닙니다. 시스템 단에서 필요한 리소스를 미리 할당해두고 필요할때마다 스케일링되는 방식이 더 근본적인 방법이라 생각합니다.

<br/>

지금의 문제는 이렇게 어느정도 보완했으나 내년에는 MWAA를 걷어내고 EKS에 airflow 직접 배포를 추진해보려고 합니다. 앞으로 필요한 리소스는 더 많아질텐데 MWAA의 비싼 가격 대비 자유도와 안정성 모두 만족스럽지 못하기 때문입니다.

회사에 DE 인원이 많은 편도 아니고 airflow는 거의 혼자서 운영하고 있긴 하지만 EKS 배포를 조금씩 준비하려고 합니다. managed 서비스는 어디까지나 인원이 부족하거나 초기에 서비스를 사용하고자 할때 간편하게 사용할수 있는 방법이고, 서비스가 커지면 이렇게 갈 수밖에 없다고 생각합니다. 실제로 GCP의 composer로 초기에 airflow를 사용했던 회사들이 규모가 어느정도 커지면 직접 개발해 사용하고 있는 사례를 여럿 보았습니다.

할 수 있는 기회가 왔을 때 잡으려면 지금부터 미리미리 준비해두는게 좋겠습니다. 다음 포스팅은 아마 Airflow를 쿠버네티스 환경에 배포하는 PoC가 될 것 같습니다.

