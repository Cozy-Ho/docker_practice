## Haproxy + Docker 로 Loadbalancing 하기

index.js라는 파일에는 간단한 node 서버 코드를 작성 했다.

```js
var http = require('http');
var os = require('os');
http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/html'});
    res.end(`<h1>I'm ${os.hostname()}</h1>`);
}).listen(8080);
```
<br>

`Dockerfile` 이라는 이름으로 파일을 만들고 아래 코드를 작성했다.

```dockerfile
FROM node
RUN mkdir -p /usr/src/app
COPY index.js /usr/src/app
EXPOSE 8080
CMD [ "node", "/usr/src/app/index" ]
```

두 파일을 같은 디렉토리에 위치시키고 다음 명령어를 실행한다.

> $ docker build -t test .

이제 `test`라는 이름으로 도커 이미지가 생성 되었다.

---

## Docker-compose 사용하기


```yaml
version: '3'

services:
  test:
   image: test
   ports:
     - 8080
   environment:
     - SERVICE_PORTS=8080
   deploy:
     replicas: 20
     update_config:
       parallelism: 5
       delay: 10s
     restart_policy:
       condition: on-failure
       max_attempts: 3
       window: 120s
   networks:
     - web

  proxy:
    image: dockercloud/haproxy
    depends_on:
      - test
    environment:
      - BALANCE=leastconn
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 80:80
    networks:
      - web
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  web:
    driver: overlay
```

`docker-compose.yaml` 이라는 이름으로 파일을 만들고 위 코드를 작성한다.

>옵션 설명

1. 첫 번째 서비스는 test Node.js 앱 이다. 조금 전에 빌드한 test 이미지로 실행된다.<br>8080 포트를 외부에 연결하고, 환경 변수로 SERVICE_PORTS로 작성했다.<br>deploy 옵션으로 20개의 복사본(replicas)을 만들고 업데이트 설정과 재시작 설정을 추가했다.<br>파일의 마지막에 작성한 network인 web 네트워크에 모든 컨테이너를 연결해둔 것이 가장 중요한 포인트.

2. 두 번째 서비스는 Docker 팀의 haproxy 이미지로 만든 HAProxy이다.<br>이미 Docker 팀에서 만들어 두었기 때문에 우리는 이미지를 빌드할 필요 없이 가져다 사용하면 된다.<br>depends_on 옵션으로 test 서비스가 부팅이 완료된 이후에 실행을 시작한다.<br>또한 volumes 옵션으로 docker.sock 파일을 공유한다.<br>HAProxy 컨테이너가 네트워크에 이미 있거나 새롭게 들어오는 컨테이너들을 찾고 확인할 수 있어야 하기 때문. 80 포트는 외부에 연결하고 web 네트워크에도 연결했다.<br>마지막으로 deploy 설정에서 manager node에서 항상 실행하도록 설정한다.<br>이건 Docker Swarm의 설정으로, node가 여러 개라면 volumes 옵션 때문에 필요하다.

3. 마지막으로 web이라는 이름의 network를 생성했다.

<br>

---

<br>

## docker swarm 사용하기

> $ docker swarm init

네트워크, 서비스, 그리고 모든 컨테이너들을 `스택(stack)`이라고 부른다. 

스택을 생성하기 위해서는 `docker stack` 명령어를 사용해야 하지만 스택을 `docker-compose.yml` 파일로 수행하고싶다. 그래야 우리가 설계한대로 진행해줄테니까.

> $ docker stack deploy --compose-file=docker-compose.yaml prod

위의 명령어를 입력한다.

`deploy` 명령으로 새로운 스택을 배포하고, `docker-compose.yaml`을 사용해 수행하기 위해서 `--compose-file` 플래그(flag)를 사용했다. 물론 이미 있는 스택을 업데이트할 때에도 명령을 사용할 수 있다. 마지막으로 스택에 `prod` 라는 이름을 붙인다. 

> $ curl http://localhost

`http://localhost` 주소로 요청을 날리면, 응답으로 컨테이너 ID를 받을 수 있다. 그러면 지금 상황에서는 매 요청마다 다른 ID를 리턴한다.

`docker service ls` 명령으로 서비스들을 확인할 수 있다. 어떤 서비스가 동작하고 있는지, 몇개의 복사본이 있는지 등을 확인할 수 있다.

<br>

---

<br>

이제 두번째 버전의 test 앱을 작성해보자. 코드를 약간 바꿔서 응답의 마지막에 느낌표를 추가했다.
<br>

```js
var http = require('http');
var os = require('os');
http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/html'});
    res.end(`<h1>I'm ${os.hostname()}!!!</h1>`);
}).listen(8080);
```
<br>

> $ docker build -t test:v2 .

위 명령어로 v2태그를 달고 새로운 test이미지를 빌드한다.

서비스의 중단없이 `prod` 스택에 `test` 서비스를 `v2`로 교체하기 위해서는

> $ docker service update --image test:v2 prod_test 

위 명령을 사용한다.

그러면 `docker-compose.yaml`에 명시한 업데이터 설정과 같이 각 5개의 컨테이너가 순차적으로 업데이트를 한다.


만약 20개의 컨테이너보다 더 많이 필요하여 스케일을 키우고 싶다면

> $ docker service scale prod_test=50

위의 명령을 수행하면 된다. 도커는 test:v2 이미지로 30개의 컨테이너를 추가로 실행할 것 이다.

---

## 서비스 중지

배포한 서비스를 중지하려면 다음 명령어를 입력한다.

> $ docker stack rm test

swarm을 종료하기 위해서는 다음 명령어를 입력한다.

> $ docker swarm leave --froce


