# S3 Sink Connector

## 환경

도커 환경을 기반으로하며, `make` 명령어를 통해 예제 파이프라인을 구성합니다.

- [docker](https://www.docker.com/)
- docker-compose
- [jq](https://stedolan.github.io/jq/)
- [make](https://www.gnu.org/software/make/manual/make.html)
- [pip](https://pypi.org/project/pip/)

## 파이프라인 구성하기

### 1. AWS CLI 설정

aws cli를 설치하고, `localstack` 에 관한 프로파일을 설정합니다.
`configure-aws-localstack` 과정에서 아래와 같은 정보를 입력합니다.

- AWS Access Key ID [None]: `user`
- AWS Secret Access Key [None]: `secret`
- Default region name [None]: `ap-northeast-2`
- Default output format [None]:

아래 명령어를 실행합니다.

- `make install-awscli`
- `make configure-aws-localstack`

```shell
$ make install-awscli
/Library/Frameworks/Python.framework/Versions/3.8/bin/pip install awscli
Collecting awscli
...
Installing collected packages: awscli

$ make configure-aws-localstack
/Library/Frameworks/Python.framework/Versions/3.8/bin/aws --endpoint-url=http://localhost:4566 configure --profile localstack
AWS Access Key ID [None]: user
AWS Secret Access Key [None]: secret
Default region name [None]: ap-northeast-2
Default output format [None]:
```

### 2. 컨테이너 구성

docker-compose.yaml 에 정의된 명세에 따라 컨테이너를 구성합니다.
구성되는 컨테이너 목록은 아래와 같습니다.

- zookeeper
- broker
- connect
- localstack

아래 명령어를 실행합니다.

- `make docker-compose-up`

```shell
$ make docker-compose-up
/usr/local/bin/docker-compose -f /Users/daehokimm/kafka-examples/s3-sink-connector/docker-compose.yaml up -d
Docker Compose is now in the Docker CLI, try `docker compose up`

Creating localstack ... done
Creating zookeeper  ... done
Creating broker     ... done
Creating connect    ... done
```

### 3. 버켓 구성

예제 환경에서 사용할 버켓을 localstack에 생성합니다.
이를 위해 아래 명령어를 실행합니다.

- `make create-demo-bucket`

```shell
$ make create-demo-bucket
/Library/Frameworks/Python.framework/Versions/3.8/bin/aws --endpoint-url=http://localhost:4566 s3 mb s3://demo-bucket --profile localstack
make_bucket: demo-bucket
```

### 4. 커넥터 배포

구성된 커넥트 컨테이너(`connect`)에 파이프라인을 구성할 커넥터를 배포합니다.
배포하는 커넥터는 아래 2가지입니다.

- `datagen-local` : 데이터를 임의로 생성하여 카프카에 프로듀싱하는 커넥터
- `s3-sink-local` : 카프카의 토픽 메시지를 컨슘하여 s3에 적재하는 커넥터

이를 위해 아래 명령어를 실행합니다.

- `make deploy-connectors`

```shell
$ make deploy-connectors
/usr/bin/curl -X PUT -H "Content-Type: application/json" -d @/Users/daehokimm/kafka-examples/s3-sink-connector/configs/connector-datagen-local.json localhost:8083/connectors/datagen-local/config | /usr/local/bin/jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3208  100  1631  100  1577   2907   2811 --:--:-- --:--:-- --:--:--  5718
{
  "name": "datagen-local",
  "config": {
...
  },
  "tasks": [],
  "type": "source"
}
/usr/bin/curl -X PUT -H "Content-Type: application/json" -d @/Users/daehokimm/kafka-examples/s3-sink-connector/configs/connector-s3-sink-local.json localhost:8083/connectors/s3-sink-local/config | /usr/local/bin/jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1977  100   995  100   982   2068   2041 --:--:-- --:--:-- --:--:--  4101
{
  "name": "s3-sink-local",
  "config": {
    ...
  },
  "tasks": [],
  "type": "sink"
}
```

생성된 커넥터 상태를 확인하기 위해 아래 명령어를 실행합니다.

- `make check-connectors`

```shell
$ make check-connectors
/usr/bin/curl localhost:8083/connectors/datagen-local/status | /usr/local/bin/jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   163  100   163    0     0   9055      0 --:--:-- --:--:-- --:--:--  9055
{
  "name": "datagen-local",
  "connector": {
    "state": "RUNNING",
    "worker_id": "connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "connect:8083"
    }
  ],
  "type": "source"
}
/usr/bin/curl localhost:8083/connectors/s3-sink-local/status | /usr/local/bin/jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   161  100   161    0     0  17888      0 --:--:-- --:--:-- --:--:-- 17888
{
  "name": "s3-sink-local",
  "connector": {
    "state": "RUNNING",
    "worker_id": "connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "connect:8083"
    }
  ],
  "type": "sink"
}
```

### 5. 오브젝트 적재 확인

메시지가 localstack의 s3에 오브젝트로 잘 적재됐는 지 확인합니다.
이를 위해 아래 명령어를 실행합니다.

- `make list-bucket`

```shell
$ make list-bucket
/Library/Frameworks/Python.framework/Versions/3.8/bin/aws --endpoint-url=http://localhost:4566 s3 ls s3://demo-bucket/backup/examples-s3-sink/year=2021/month=06/day=28/hour=07/ --profile localstack
2021-06-28 07:01:00      47896 examples-s3-sink+0+0000095403.json
2021-06-28 07:02:00      50082 examples-s3-sink+0+0000095649.json
2021-06-28 07:03:00      48268 examples-s3-sink+0+0000095906.json
2021-06-28 07:04:00      44800 examples-s3-sink+0+0000096154.json
2021-06-28 07:05:00      47458 examples-s3-sink+0+0000096384.json
2021-06-28 07:06:00      46114 examples-s3-sink+0+0000096628.json
```

적재된 오브젝트를 다운로드 받으려면 아래 명령어를 실행합니다.

- `make download-object`

```shell
$ make download-object
Object name : examples-s3-sink+0+0000098745.json
download: s3://demo-bucket/backup/examples-s3-sink/year=2021/month=06/day=28/hour=07/examples-s3-sink+0+0000098745.json to data/examples-s3-sink+0+0000098745.json
```

다운로드 된 오브젝트 내용은 `data` 디렉토리 하위에서 확인할 수 있습니다.
