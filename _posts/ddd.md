---
title: "[spark] Docker Ubuntu 컨테이너로 Spark 실습환경 만들기 2. master - worker 환경 구성"
date: 2021-05-09 00:00:00
categories:
- spark
tags: [spark, docker]

---

# worker 컨테이너

master용 컨테이너를 만들었으니, worker용으로 사용할 컨테이너를 하나 띄워보겠습니다. worker는 더 늘려도 관계없고, 꼭 컨테이너를 띄워 독립적인 환경에서 구성하지 않아도 됩니다.

이 부분이 새로 알게 된 사실인데, master와 worker는 각각 독립적인 서버 환경에서 구축되어야 한다고 막연하게 생각했었는데 꼭 그럴 필요는 없다는 사실입니다. 

[spark 공식문서](https://spark.apache.org/docs/3.1.1/spark-standalone.html)서도 다음과 같이 설명하고 있습니다.
> In addition to running on the Mesos or YARN cluster managers, Spark also provides a simple standalone deploy mode. You can launch a standalone cluster either manually, by starting a master and workers by hand, or use our provided launch scripts. It is also possible to run these daemons on a single machine for testing.

테스트 / 실습 단계에서는 굳이 머신을 분리할 필요가 없다는 게 요지입니다. 그래도 한 번 띄워보겠습니다. 따로 설정하지 않으면 worker web ui는 8081 포트로 바인딩되기 때문에 8081 포트만 하나 열어주겠습니다.

```python
docker run -itd --name spark-worker -p 8081:8081 hadoop-spark
```
<br/>

# ssh
사실 도커 컨테이너 내부 ssh 통신을 권장하지는 않습니다 ([공식 문서](https://docs.docker.com/samples/running_ssh_service/))). 컨테이너끼리 자체적인 [network](https://docs.docker.com/network/)를 제공하고 있습니다.
그러나 docker network가 spark 환경에서 원하는대로 잘 작동하는지 테스트해보지 않았고, 책에서도 ssh를 이용해 서버를 연결했기 때문에 ssh를 사용하겠습니다.

master container에 접속한 뒤 ssh key를 생성해줍니다.
```shell
docker exec -it spark-master bash

root@master# ssh-keygen -t rsa
...
생략
...
```
이그럼 이제 worker 컨테이너에 key를 등록하겠습니다.
```shell
docker exec -it spark-worker bash

root@worker# mkdir ~/.ssh
root@worker# cd ~/.ssh
root@worker# vi authorized_keys
```
~/.ssh 폴더 안에 autorized_keys라는 파일을 생성한 뒤 master의 id_rsa.pub 파일 내용을 복사해 붙여넣습니다. master의 public key를 이 곳에 등록함으로써 master가 ssh를 이용해 worker에 접속할 수 있게 됩니다.

worker도 master와 동일하게 ssh key를 만들어 master의 authorized_keys에 등록합니다.

등록이 완료되었다면 master 컨테이너에서 worker 컨테이너에 접속이 가능한지 확인해봅니다. worker container IP는 컨테이너 내부 `/etc/hosts`파일에서 확인할 수 있습니다.



```shell

root@master# ssh ${worker container IP}
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.10.25-linuxkit x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Mon May 10 00:23:16 2021 from ${master container IP}
root@worker:~#
```
<br/>

# worker 등록
master에 worker를 등록합니다. master 컨테이너의 ${SPARK_HOME}/conf로 들어간 뒤, workers.template 파일을 slaves 파일로 복사합니다.
```shell

root@master# cd ${SPARK_HOME}/conf
root@master:~# cp workers.template workers
```
정확한 버전은 모르겠으나 3.0부터는 slave라는 단어 대신 worker라는 단어를 사용하고 있습니다. conf 폴더 안에 설정 파일들이 template 파일로 존재하는데 버전마다 그 이름은 slave / worker로 다를 수 있습니다.

파일을 생성했다면 workers 파일 안에 적혀있는 localhost는 지워주고 아까 확인했던 worker 컨테이너의 ip를 적어줍니다.

그럼 이제 worker 등록까지 완료되었으니 클러스터 매니저를 실행해보겠습니다.
```shell

root@master# cd ${SPARK_HOME}/sbin
root@master# ./start-all.sh
starting org.apache.spark.deploy.....
```

localhost:8080에 접속해 master web ui를 확인했을 때, worker에 뭔가 제대로 등록되어 있다면 성공!
![image](https://user-images.githubusercontent.com/52685258/117578295-9ade4c00-b128-11eb-903d-91f89be90ba0.png)

