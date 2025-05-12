# Operator의 init 메서드를 조심하세요!

이직 후 약 1년만에 쓰는 게시글입니다. 정신없이 지내다보니 어느새 1년이 지났네요.

airflow 관련해서 좀 더 큰 프로젝트를 하나 하고있어서 관련된 포스팅을 정리하고 있는데, 와중에 사고를 하나 친게 있어서 배운 점을 공유할까 합니다. 요지는 airflow operator의 init 메서드를 잘못 쓰면 큰 장애로 이어질 수 있다는 것입니다.



# problem

새벽에 갑자기 문제가 발생했습니다.

![img1](https://github.toss.bz/storage/user/1496/files/bad5cf70-a5f3-4c8b-93f0-644c24229fca)

db connection이 점진적으로 증가해 감소하지 않았고, 노드를 강제로 재시작하며 커넥션을 끊었지만 connection의 증가는 멈추지 않았습니다. DB가 OLAP였기에 망정이지, OLTP에서 동일한 증상이 있었다면 긴급으로 장애 원인을 파악해야 하는 상황이었습니다.



문제가 된 부분은 여기였습니다. 코드를 다 공개하기는 어려워 요약한 코드로 설명합니다.

```python
def get_connection():
  host = 'host'
  port = 'port'
  user = 'user'
  password = 'password'
  
  return connect(host=host, port=port, user=user, password=password)

class CustomOpearator(BaseOperator):
  def __init__(self):
    self.connection = BaseHook.get_connection('mydb')
    
  def run(self):
    query = "SELECT * FROM db.table"
    with closing(self.connection.cursor()) as cur:
      cur.execute(query)
```

특정 DB에 쿼리를 하기 위한 operator였고, Operator가 생성될 때 db와 connection을 맺습니다. airflow에는 기본 hook이 제공되고 provider를 따로 설치하면 웬만한 db와 통신하기 위한 hook을 제공합니다. 그러나 이번 경우는 원하는 기능에 딱 들어맞는 Hook이 없어서 BaseHook을 사용해 airflow에 저장해놓은 Connection 정보만 가져와 connect 객체를 생성하고, run 함수에서 cursor를 가져와 쿼리를 실행하도록 했습니다. 

closing을 사용해 context가 종료되면 cursor 또한 닫도록 했습니다. 장애 시에도 이 operator를 의심했으나 cursor에 closing이 되어있어 원인이 아닐거라 생각했습니다.



# 원인

원인은 init 메서드에서 발생하는 connection 자체에 있었습니다. Operator의 init 메서드는 task instance가 만들어질때 실행됩니다. 즉, dag가 파싱되는 시점에 init은 실행됩니다.

이런 문제를 방지하기 위해, 대부분 Hook providers들은 run이나 execute 메서드에서 DB 커넥션을 맺습니다. 커스텀 오퍼레이터를 만들때에도 이런 점을 조심해서 execute() 메서드에서 커넥션을 맺도록 해야겠습니다.



