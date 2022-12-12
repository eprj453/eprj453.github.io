---
title: "[Airflow] Airflow on Kubernetes - 1. Minikube"
date: 2022-11-21 00:00:00
categories:
- airflow
tags: [airflow, aws, kubernetes]

---

앞선 포스팅에서 언급했듯, 현업에서 managed airflow의 2가지 문제점에서 한계를 느꼈습니다.

- 운영상에 필요한 사항들을 정확히 파악하기 어렵다(블랙박스).
  - 각종 컴포넌트(worker, db...)들에 직접 접근할 수 없어 정확한 에러 원인을 파악하기가 어렵다.
- 컴포넌트들의 커스텀을 하기가 어렵다.
  - MWAA의 로깅은 무조건 aws cloudwatch를 이용한다. aws의 또 다른 서비스가 mwaa의 성능에 문제가 되더라도 이를 변경할 수 없다.
  - 여러 줄의 log를 한번에 I/O 하려고 하면 task 실행시간 자체가 늘어나버리는 이슈

<br/>

이에 직접 개발한 airflow의 필요성을 다시 느꼈고, 이전에 사내 물리 서버에서 사용했던 Docker + CeleryExecutor 조합으로 배포하려고 했으나, 이 또한사용하면서 문제점을 느꼈던 조합입니다.

- Queue에 많은 task가 몰릴 때, scale-in으로 다수의 worker를 구성해 소화시킬 수 있는가?
  - 몇백개의 task가 순식간에 몰리는 순간에 task switch가 상당히 느려졌던 경험이 있음. worker 스케일을 유연하게 관리하는 데에는 한계가 있었음.
- 문제가 생겨 worker가 죽는다면, 거기에 subprocess로 물려있던 task들은?
  - SPOF를 해결해줄 수 없는 태생적 한계

<br/>

이를 해결하기 위해 airflow를 사용하는 많은 기업들이 Kubernetes 환경에서 airflow를 사용하고 있습니다. Kubetnetes 환경의 airflow에서는 CeleryExecutor에서 제가 경험했던 2가지 문제점을 해결할 수 있을 뿐만 아니라, 지금보다 더 유연한 task 관리가 가능하다는 생각이 들었습니다.

- task 별로 리소스를 다르게 할당해 경중에 따른 리소스 부여 가능
- task들이 pod 단위로 뜨기 때문에 CeleryExecutor가 갖는 SPOF에서 비교적 자유로움



현재는 회사에 DE 인력이 거의 없어 혼자서 이를 개발 + 배포 + 운영하기는 어렵겠지만, managed airflow의 한계가 생각보다 빠르게 다가오고 있는 것을 느끼고 있고 그 때 대비하려고 하면 너무 늦을것 같다는 생각이 들어 미리 준비하는 차원에서 airflow on kubernetes + kubernetesExecutor 조합의 직접 개발 + 배포를 준비하고 있습니다. 

물론 쿠버네티스가 만능은 아닙니다. 러닝 커브가 상당한만큼 쿠버네티스를 이해하지 못해 정작 필요한 airflow가 버벅거리는 주객전도의 상황이 펼쳐질 가능성 또한 높습니다.

그러나 managed airflow를 약 1년간 운영하며 느낀 문제점들을 해소할 수 있는 방안은 쿠버네티스라고 생각했고, 이대로 가다가는 airflow가 오롯이 기술 부채로 남을 것 같아서 할 수 있는 것들을 조금씩 해놓으려 합니다.

비슷한 문제를 경험하고 계신 분들에게 조금이나마 도움이 되길 바라며, minikube -> EKS or GKE 배포 순으로 진행합니다.

이 포스팅에서는 kubernetes에 대한 자세한 설명은 하지 않습니다. 개인적으로 도움받았던 [inflearn - 초보를 위한 쿠버네티스 안내서](https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EC%9E%85%EB%AC%B8) 링크를 첨부합니다. 데브옵스 엔지니어가 아닌 데이터 ETL 개발자가 airflow on kubernetes를 진행하며 가져야 할 기초 소양은 전부 들어있다고 생각합니다.



<br/>

# Minikube

minikube는 로컬 환경에서 쿠버네티스 클러스터를 사용해볼 수 있도록 해줍니다. minikube를 위한 도커 컨테이너가 뜨고, 컨테이너 내부에서 쿠버네티스 클러스터 환경이 구성되어 있습니다.

<img width="716" alt="image" src="https://user-images.githubusercontent.com/52685258/202919320-fe70ffbd-08f4-4282-b054-68a6b7e065ac.png">

물론 로컬에서 사용하는만큼 리소스 사용은 상당히 제한적입니다. 실제 운영보다는 테스트 용도에 적합하고, 사실 어느정도 크게 테스트해보려 하면 테스트도 힘들어졌습니다. 로컬에서 간단하게 쿠버네티스 환경을 써볼 수 있다는 정도로만 사용하면 좋을 것 같습니다.

[설치 가이드](https://minikube.sigs.k8s.io/docs/start/)에서 설치하거나, MAC에서는 brew를 사용해 설치도 가능합니다.

```shell
brew install minikube
```

<br/>

# helm chart

쿠버네티스의 오브젝트들은 yaml 파일에 그 리소스를 정의합니다. 이와 같은 파일들을 manifest 파일이라고 합니다.

```yaml
#webserver.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webserver
spec:
  replicas: 10
  selector:
    matchLabels:
      app: webfront
    template:
      metadata:
        labels:
          app: webfront
      spec:
        containers:
        - image: nginx
          name: webfront-container
          ports:
            - containerPort: 80
```

이런 manifest 파일은 경우에 따라 여러개가 될 수 있기 때문에 이것들을 통합 배포할 수 있도록 해주는 파일이 helm chart입니다. helm chart 또한 별도의 파일들로 구성되어 있으며, 자세한 내용은 [helm Docs](https://helm.sh/ko/docs/topics/charts/)를 참고하시면 좋을 것 같습니다.

airflow에서는 [astronomer](https://www.astronomer.io/)에서 제공하는 공식 helm chart와 user community에서 오픈소스로 개발하고 있는 helm chart 2가지가 일반적으로 사용됩니다. user comminuity의 차트도 여러 회사에서 상용에 사용할만큼 역사도 오래됐고(since 2018 ~ ) 구성이 잘 되어 있어서 둘 중 하나를 선택하면 될 것 같습니다. 저는 user community 차트를 사용했습니다.

<br/>

# minikube에 helm chart로 배포하기

해당 테스트는 user community의 [quickstart](https://github.com/airflow-helm/charts/blob/main/charts/airflow/docs/guides/quickstart.md )를 따라 진행했고, helm에서 사용할 value.yaml은 샘플로 제공해주는 [**sample-values-KubernetesExecutor.yaml**](https://github.com/airflow-helm/charts/blob/main/charts/airflow/sample-values-KubernetesExecutor.yaml)을 사용했습니다.

```yaml
## add this helm repository
helm repo add airflow-stable <https://airflow-helm.github.io/charts>

## update your helm repo cache
helm repo update

## set the release-name & namespace
export AIRFLOW_NAME="airflow-cluster"
export AIRFLOW_NAMESPACE="airflow-cluster"

## create the namespace
kubectl create ns "$AIRFLOW_NAMESPACE"

## install using helm 3
helm install \\
  "$AIRFLOW_NAME" \\
  airflow-stable/airflow \\
  --namespace "$AIRFLOW_NAMESPACE" \\
  --version "8.X.X" \\
  --values ./sample-values-KubernetesExecutor.yaml
```

배포에 큰 문제는 없었으나 만약 문제가 생긴다면

- 내 default 클러스터가 minukube가 맞는지
- 내 defalut namespace가 airflow-cluster가 맞는지
- namespace를 재사용하려 하지 않는지

확인해보시기 바랍니다.

helm이 정상적으로 리소스를 배포한다면 다음과 같이 airflow에 필요한 모든 컴포넌트가 뜹니다.

![image](https://user-images.githubusercontent.com/52685258/203100833-2274385f-4783-4ad2-b8dc-5b52a83f57e4.png)

그런데 브라우저에서 webserver에 바로 접속할 수 있는 방법이 없습니다. 로드밸런서나 Ingress를 두는 것이 일반적이라 방법이지만, 테스트 용도이기 때문에 webserver용으로 떠있는 service 오브젝트에 포트 포워딩만 해주어 외부에서 해당 포트로 진입할 수 있게끔 해주도록 하겠습니다.

```shell
kubectl port-forward svc/${AIRFLOW_NAME}-web 7070:8080 --namespace $AIRFLOW_NAMESPACE
```

![image](https://user-images.githubusercontent.com/52685258/203101690-cc8f8c31-7444-4b50-bf3f-86c5427f2831.png)

<br/>

이제 127.0.0.1:7070을 통해 minikube에 8080으로 떠있는 webserver에 진입할 수 있습니다.

![image](https://user-images.githubusercontent.com/52685258/203101928-d4a47c55-cd97-4e55-abda-e0d5b3133a91.png)

<br/>

# Next

이대로는 당연히 상용에서 사용할 수 없습니다. 당장 생각나는 것만 해도

- replica는 어떻게 운영할 것인가?
- 소스, 배포 관리를 어떻게 할 것인가?
- 외부 저장소는 어떻게 할 것인가?
- 보안은 어떤 식으로 구성할 것인가?

정도입니다.

eks에 배포하기 앞서 필요한 리소스들을 살펴보고, helm chart를 구성해보도록 하겠습니다. 

