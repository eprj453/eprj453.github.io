---
title: "[AWS] Kinesis 도입기 2. lambda"
date: 2021-08-18 00:00:00
categories:
- AWS
tags: [AWS]
---



firehose에서는 Lambda를 통해 데이터를 가공하거나, glue data catalog를 통해 데이터 타입 검증, 포맷 변경이 가능합니다. 

<br/>



# lambda

Lambda 함수를 통해 firehose로 들어온 데이터를 가공할 수 있습니다. Firehose로 직접 쏘든, data stream에서 데이터를 받든 형태는 2가지이고, 받는 방식은 동일합니다.

- 단일 레코드
- 2개 이상의 레코드

제가 사용했던 Lambda입니다.

```python
import awswrangler as wr
import pandas as pd
import numpy as np
import json
import base64
import time
from pprint import pprint

# 모든 데이터의 타입을 string(object)로 변경합니다.
def type_to_object(obj):
    if type(obj) == list:
        return [str(i) for i in obj]
    else:
        return str(obj)

# {a:{b:1, c:2}} 형태의 dictionary를 {a.b:1, a.c:2} 형태로 편 뒤 json 형태로 dump합니다.
def flatten_json(nested_json: dict):
    flat_json = {}

    for key, value in nested_json.items():
        if value and type(value) == dict:
            for inside_key, inside_value in value.items():
                merged_key = f'{key}.{inside_key}'
                flat_json[merged_key] = json.dumps(type_to_object(inside_value))
        else:
            flat_json[key] = json.dumps(type_to_object(value))

    return flat_json


# dictionary를 json으로 dump한 뒤 base64로 인코딩합니다 
def dict_to_bytes(value):
    json_to_str = json.dumps(value)
    return base64.b64encode(json_to_str.encode('utf-8'))

  
# return value 형태를 잡아줍니다.
def generate_result_value(dict_bytes, record_id):
    return {
        'data': dict_bytes,
        'recordId': record_id,
        'result': 'Ok'
    }
    
    

def lambda_handler(event, context):
    records = []
    
    for record in event['records']:
        record_id = record['recordId']
        payload = base64.b64decode(record['data']).decode('utf-8')
        nested_json_list = json.loads(payload)
        flat_dict_list = [flatten_json(nested_json) for nested_json in nested_json_list]
    
        for flat_dict in flat_dict_list:
            dict_byte = dict_to_bytes(flat_dict)
            result_value = generate_result_value(dict_byte, record_id)
            records.append(result_value)
        
    return {
        'records': records
    }
    
    
```





lambda_handler 함수 -> event 인자 -> records 안에 firehose에서 넘겨받은 데이터가 들어있습니다. 이것을 원하는 형태로 가공해 다시 return해야 합니다. 

record는 Bytes 코드로 들어오기 때문에 디코딩 처리가 필요합니다. 디코딩 처리가 된 payload는 str이 아닌 dictionary / list 형태로 처리가 가능합니다.

```python
def lambda_handler(event, context):
    for record in event['records']:
        data = record['data']
        payload = base64.b64decode(data).decode('utf-8')
```



Record 하나가 벌크로 묶은 json list인 경우, 처리 이후에도 record 안에서 for loop를 돌며 return value 형식으로 값을 넣어줍니다.

```python
record_id_idx = 0
for flat_dict in flat_dict_list:
  dict_byte = dict_to_bytes(flat_dict)
  result_value = generate_result_value(dict_byte, record_id + str(record_id_idx) if record_id_idx != 0 else record_id)
  records.append(result_value)
  record_id_idx += 1
```



<br/>

## lambda 주의점

lambda에서 제공하는 blueprint에서 firehose용 템플릿도 제공하고 있습니다. 이걸 사용하지만 웬만한 커스텀은 어렵지 않을것입니다. 아래 공식문서에서 lambda blueprint 파트를 참고해주세요.

[https://docs.aws.amazon.com/ko_kr/firehose/latest/dev/data-transformation.html](https://docs.aws.amazon.com/ko_kr/firehose/latest/dev/data-transformation.html)



### return value 형식

- firehose 내에서 Lambda를 거칠 때, Lambda의 return value 형식이 고정되어 있습니다. 형식에 맞지 않을 경우 processing-failed 에러를 발생시킵니다. 자세한 내용은 공식 문서를 참고해주세요.

  https://docs.aws.amazon.com/firehose/latest/dev/data-transformation.html#data-transformation-failure-handling

```python
return {
  'records' : [
     	{
        'data': data,
        'recordId': record_id,
        'result': 'Ok'
      },
      {
        'data': data,
        'recordId': record_id,
        'result': 'Ok'
  		}
    ...
    ...
    ...
    ...
					]
  }
```



위  List 형태의 객체를 넘기되, `records`라는 key를 가진 딕셔너리를 반환해야 합니다.

firehose에서 s3 error backup을 해두면, 에러가 났을때 아래와 같이 에러 메세지와 원본 데이터가 담긴 파일을 떨궈줍니다. return 형식이 맞지 않거나 key 이름이 다른 경우 발생하는 에러 메세지 일부입니다.

```json
{
  "attemptsMade":4,
  "arrivalTimestamp":1627921286181,
  "errorCode":"Lambda.FunctionError",
  "errorMessage":"The Lambda function was successfully invoked but it returned an error result.",
 "attemptEndingTimestamp":1627921366634,"rawData":"YidbeyJhcGlfbGV2ZWwiOjMwLCJhcGlfcHJvcGVydGllcyI6eyJhbmRyb2lkQURJRCI6Ijg5MzExNjgyLTIxNDQtNDc5OC04OTZiLTZhNTBjMTUyZTdhZiIsImdwc19lbmFibGVkIjp0cnVlLCJsaW1pdF9hZF90cmFja2luZ
  ....
  ....
  ....
```

<br/>

### record_id 중복

- return으로 들어가는 records마다 존재하는 recordId는 중복되면 안됩니다.

사실 이 사항은 record 형태에 따라 문제가 될수도, 되지 않을수도 있겠으나 맨 처음 lambda에서 `event['records']` 에 있는 원본 데이터의 형태가 이런 식이라면 문제가 될 수 있습니다.

```python
event['records'] = [
  [{}, {}, {}, {}], # record
  [{}, {}, {}, {}, {}], # record
  [{}, {}, {}, {}, {}], # record
  ...
  ...
  ...
]
```

record가 object list인 경우입니다. 이 경우에는 record를 다시 풀어서 각각 하나의 object로 보고 처리를 해야하는데, 이 때 record id가 문제가 될 수 있습니다. object 처리를 잘못 할 경우 아래와 같이 record_id가 중복될 수 있습니다. 임의의 데이터와 record_id가 있다고 가정해본다면 이런 상황입니다.

```python
event['records'] = [
  [{1}, {2}, {3}, {4}], # record (record_id = 1)
  [{5}, {6}, {7}, {8}, {9}], # record (record_id = 2)
  [{10}, {11}, {12}, {13}, {14}], # record (record_id = 3)
  ...
  ...
  ...
]


return {
  'records': [
    {data:1, record_id: 1},
    {data:2, record_id: 1},
    {data:3, record_id: 1},
    {data:4, record_id: 1},
    {data:5, record_id: 2},
    ...
    ...
    ...
  ]
}
```



위에서 제가 사용했던 예시 Lambda도 저대로 바로 쓰면 에러가 일어납니다. record_id의 중복을 피하기 위해 다음과 같이 로직을 변경했습니다.

예시 Lambda 함수에 이 로직이 들어있는 것이 이것 때문입니다. Input과 output의 record_id가 꼭 같을 필요는 없습니다. output에 들어갈 record_id를 각각 다르게 주기 위한 방법입니다.

```python
	import awswrangler as wr
import pandas as pd
import numpy as np
import json
import base64
import time
from pprint import pprint

# 모든 데이터의 타입을 string(object)로 변경합니다.
def type_to_object(obj):
    if type(obj) == list:
        return [str(i) for i in obj]
    else:
        return str(obj)

# {a:{b:1, c:2}} 형태의 dictionary를 {a.b:1, a.c:2} 형태로 편 뒤 json 형태로 dump합니다.
def flatten_json(nested_json: dict):
    flat_json = {}

    for key, value in nested_json.items():
        if value and type(value) == dict:
            for inside_key, inside_value in value.items():
                merged_key = f'{key}.{inside_key}'
                flat_json[merged_key] = json.dumps(type_to_object(inside_value))
        else:
            flat_json[key] = json.dumps(type_to_object(value))

    return flat_json


# dictionary를 json으로 dump한 뒤 base64로 인코딩합니다 
def dict_to_bytes(value):
    json_to_str = json.dumps(value)
    return base64.b64encode(json_to_str.encode('utf-8'))

  
# return value 형태를 잡아줍니다.
def generate_result_value(dict_bytes, record_id):
    return {
        'data': dict_bytes,
        'recordId': record_id,
        'result': 'Ok'
    }
    
    

def lambda_handler(event, context):
    records = []
    
    for record in event['records']:
        record_id = record['recordId']
        payload = base64.b64decode(record['data']).decode('utf-8')
        nested_json_list = json.loads(payload)
        flat_dict_list = [flatten_json(nested_json) for nested_json in nested_json_list]
    
        record_id_idx = 0 ####### 변경사항 #######
        for flat_dict in flat_dict_list:
            dict_byte = dict_to_bytes(flat_dict)
            result_value = generate_result_value(dict_byte, record_id + str(record_id_idx) if record_id_idx != 0 else record_id)
            records.append(result_value)
            record_id_idx += 1
            
    return {
        'records': records
    }
  
```

<br/>

### 최대 호출 가능한 lambda 갯수

- data stream을 Firehose의 source로 사용할 경우 고려해야 할 점입니다. Lambda는 1 shard 당 5개까지만 뜰 수 있습니다. 즉, 하나의 lambda 함수가 데이터를 처리하고 내려가는 속도보다 firehose의 lambda buffer가 차는 속도가 더 빨라 5개 이상의 lambda가 뜨는 순간, lambda는 죽어버립니다. 
- 저의 경우 cloudwatch에서 다음과 같은 에러 로그를 확인할 수 있었습니다. 에러 로그나 상황은 다 다르겠지만, shard를 적게 잡고 firehose + lambda를 사용중이시라면 shard 갯수를 늘려보시기 바랍니다.

```python
[ERROR] JSONDecodeError: Expecting value: line 1 column 1 (char 0)
Traceback (most recent call last):
  File "/var/task/lambda_function.py", line 51, in lambda_handler
    nested_json_list = json.loads(payload)
  File "/var/lang/lib/python3.8/json/__init__.py", line 357, in loads
    return _default_decoder.decode(s)
  File "/var/lang/lib/python3.8/json/decoder.py", line 337, in decode
    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
  File "/var/lang/lib/python3.8/json/decoder.py", line 355, in raw_decode
    raise JSONDecodeError("Expecting value", s, err.value) from None
```

