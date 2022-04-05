---
title: "[Airflow] Executor Deep Dive 2. CeleryExecutor"
date: 2022-04-04 00:00:00
categories:
- airflow
tags: [airflow]

---



Executor Deep Dive 2번째 파트 CeleryExecutor입니다. 사실 LocalExecutor는 production 환경에서 사용을 권장하지 않습니다. 그러나 CeleryExecutor부터는 production 환경 사용도 권장하고 있습니다.

> CeleryExecutor is recommended for production use of Airflow. It allows
> distributing the execution of task instances to multiple worker nodes. 
>
> Celery is a simple, flexible and reliable distributed system to process
> vast amounts of messages, while providing operations with the tools
> required to maintain such a system.
>
> -- CeleryExecutor Docstring



LocalExecutor보다 내용이 조금 많습니다. 달려보겠습니다!



# Class CeleryExecutor

![image-20220404085723731](/Users/psw/Library/Application Support/typora-user-images/image-20220404085723731.png)



















