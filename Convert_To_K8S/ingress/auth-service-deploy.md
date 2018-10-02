## Auth service 배포
#### Auth service 정의

기존 Zuul 서비스에 구현한 Authentication/Authorization 실행 로직을 Auth Service로 분리하여 구현하였습니다.

docker.sds-act.com/auth 이미지를 Service로 만들어 실행합니다.

```bash
$ kubectl run auth --image=docker.sds-act.com/auth --port=9000
$ kubectl get pod
$ kubectl expose pod [auth pod name] --type=NodePort --name=auth
```

#### Auth service 생성 및 접속

Auth Service가 정상적으로 동작하는지 확인해봅시다.

요청을 실행하기 전에 `auth` Pod의 실행 로그를 볼 수 있도록 준비합니다.

```bash
$ kubectl get pods  # auth pod name 확인
$ kubectl logs -f [auth pod name]
```

브라우저나 쉘을 이용해 Nginx 서버에 tags 목록을 가져오는 API `GET /api/v1/tags`를 실행 합니다.

```bash
$ curl http://<Nginx NodeIP>:<Nginx NodePort>/api/v1/tags
```

위 REST API 요청 시 `Cookie: NDE-AUTH-TOKEN=`를 Header로 포함하지 않았기 때문에 아래와 같은 실행 결과를 확인할 수 있습니다.

```text
<html>
<head><title>401 Authorization Required</title></head>
<body bgcolor="white">
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.15.2</center>
</body>
</html>
```

로그인 후 얻은 `NDE-AUTH-TOKEN` 값을 이용하여 다음 명령을 실행합니다.

```bash
$ curl -i -XPOST -H 'Content-Type: application/json; charset=utf-8' -d '{"userId":"jacky.shin", "password":"MXEydzNlNHI1dA=="}' http://<Nginx NodeIP>:<Nginx NodePort>/api/v1/signin  # NDE-AUTH-TOKEN 값 확인

$ curl -i -H "Cookie: NDE-AUTH-TOKEN=<NDE-AUTH-TOKEN value>" http://<Nginx NodeIP>:<Nginx NodePort>/api/v1/tags
```
유효한 `NDE-AUTH-TOKEN` 값을 이용해서 호출 했다면 다음과 같은 결과를 얻을 수 있습니다.

```
HTTP/1.1 200
Server: nginx/1.15.3
Date: Fri, 28 Sep 2018 04:57:35 GMT
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
X-Application-Context: admin-service:apit:5000

{
  "tags": [
    {
      "id": 2,
      "name": "2018",
      "tenantId": 1,
      "archived": false
    },
    ...
    {
      "id": 8,
      "name": "show window",
      "tenantId": 1,
      "archived": false
    }
  ]
}
```
