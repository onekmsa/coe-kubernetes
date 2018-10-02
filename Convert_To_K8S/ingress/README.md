# Netflix Zuul & Eureka Migration to Ingress

## Table of Contents
- Kubernetes Ingress 소개
- Zuul 기능을 Ingress로 마이그레이션하는 방법
- 마이그레이션 실행 및 결과 확인

## 1. Kubernetes Ingress 소개

### 기본

Ingress는 아래 그림과 같이 Kubernetes cluster로 들어오는 외부 사용자 요청을 받고 적절한 Service endpoint로 라우팅 합니다.

```
  internet
      |
 [ Ingress ]
---|-----|---
 [ Services ]
```

Ingress 기능을 활용하기 위해서는 다음 두 가지 오브젝트를 정의해야 합니다.

|                           | 역할                                                                                        |
|---------------------------|---------------------------------------------------------------------------------------------|
| Ingress Resource          | Ingress가 어떤 요청을 어떤 어플케이션으로 라우팅할지 hostname, path 정보 등을 기반으로 매핑 |
| Ingress Controller 구현체 | Ingress Resource에 정의한 규칙을 기반으로 요청을 실행하는 인스턴스                          


### Service Discovery와 Load Balancing 구현
사용자 요청을 받아 처리하는 어플리케이션을 Pod의 묶음인 Service 타입으로 정의하면 Zuul과 Eureka를 함께 사용하는 효과를 낼 수 있습니다.


## 2. Zuul 기능을 Ingress로 마이그레이션하는 방법

### Netflix OSS를 이용한 환경

아래 그림에서 Zuul은 Nginx로부터 사용자 요청을 받습니다.

그 다음 이 요청을 Zuul configuration에 정의된 라우팅 규칙에 따라 적절한 백엔드 서비스로 포워딩 합니다.

이 때 Eureka를 사용한다면 Zuul은 최근 갱신된 백엔드 서비스의 IP와 Port 정보를 받아볼 수 있습니다.

```
   Web Browser
        |
    [ Nginx ]
        |
 [ Zuul & Eureka ]
----|--------|-----
   [ Services - Auth, Admin, Content ]
```

### Nexshop Marketing Zuul 서비스 기능
현재 Nexshop Marketing Zuul 서비스가 제공하는 기능은 다음과 같습니다.

```text
a. Request Filtering & Forwarding
b. Authentication & Authorization 확인
```
### 마이그레이션: Ingress를 이용한 환경

아래 그림과 같이 Zuul/Eureka를 Ingress Resource/Controller로 전환하였습니다.

이 때 Auth 관련 비즈니스 로직은 새로운 어플리케이션으로 분리해야 합니다.

새로 생성한 Auth Service를 Ingress Resource와 연결시키면 Zuul 서비스 기능을 모두 충족시킬 수 있습니다.

```
   Web Browser
        |
    [ Nginx ]
        |
 [ Ingress Controller ]  -- uses -->  [ Inress Resource ]
----|--------|-----
   [ Services - Auth, Admin, Content ]
```


## 3. 실행 및 결과 확인

### Ingress 환경 구성 미리 보기
각 구성 요소를 생성하기 위한 관련 파일과 내용은 다음과 같습니다.

|                    |                             파일 이름                             |                                                                       설명                                                                       |
|:------------------:|-----------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
|        Nginx       | ui/ui-svc.yml, ui/ui-deployment.yml                               | Nexshop UI 소스를 포함한 Nginx 이미지를 컨테이너로 실행하는 Service definition                                                                   |
|    Ingress Resource    | ingress/nxs-ingress-rule.yaml, ingress/nxs-ingress-omit-rule.yaml | Cluster에서 처리할 Inbound traffic을 Path 기반으로 정의한 Ingress definition                                                                     |
| Ingress Controller | ingress/nginx-ingress-controller.yaml                             | Nginx를 통해 전달받은 Inbound traffic을 Ingress Resource에 따라 필터링하고 Service로 포워딩하는 작업을 담당하는 Nginx Ingress Controller definition  |
|    Auth Service    | auth/auth-svc.yaml, auth/auth-deployment.yaml                     | User request를 Service로 포워딩 하기 전에 실행할 Authentication/Authorization 확인 로직을 담은 백엔드 Service Definition |                                           |

다음과 같은 순서로 Kubernetes 리소스 생성 방법을 알아보겠습니다.
- Ngix
- Ingress Resource
- Ingress controller
- Auth Service

### Nginx

#### Nginx Service 정의

아래 definition은 크게 두가지 내용으로 구성됩니다.
- label `app=nxs-ui`를 가진 Pod을 생성하는 Nginx `Deployment`
- label `app=nxs-ui`를 가진 Pod을 Nginx `Service`로 노출

`docker.sds-act.com/nxs-ui-test` 이미지 파일은 nginx 기본 이미지 위에 Nexshop UI static 파일 소스를 포함하고 있습니다.

이렇게 실행되는 개별 Pod의 IP는 수시로 바뀔 수 있기 때문에 서비스 클라이언트는 안정적으로 백엔드 서비스를 호출할 수 없습니다.

서비스 클라이언트가 간단한 서비스 이름과 정해진 Port를 이용해 백엔드 서비스를 호출할 수 있도록 `Service` 타입으로 Pod을 그룹화하고 type을 `NodePort`로 선언합니다.

NodePort를 선언하면 `<NodeIP>:<NodePort>`로 외부에서 접근할 수 있고 NodePort는 자동으로 생성됩니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nxs-ui
  name: nxs-ui
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nxs-ui
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nxs-ui
  labels:
    app: nxs-ui
    version: v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nxs-ui
        version: v1
    spec:
      terminationGracePeriodSeconds: 5
      volumes:
      - name: nginx
        configMap:
          name: nginx
      containers:
      - name: nginx
        image: docker.sds-act.com/nxs-ui-test
        resources:
          limits:
            cpu: 1
            memory: 512Mi
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        volumeMounts:
        - name: nginx
          mountPath: /etc/nginx/servers
          readOnly: true
```

#### Nginx Service 생성 및 접속

다음 명령을 통해 Service를 생성합니다.

```bash
$ kubectl create -f ui-svc.yml
```

Nginx가 정상적으로 화면을 서비스 하는지 확인해봅시다.

```bash
$ kubectl get pods  # nginx가 실행중인 pod 확인
$ kubectl get pods [nginx pod name] -o wide  # pod이 실행중인 NodeIP 확인
$ kubectl get svc nxs-ui # NodePort 확인
$ curl http://<Nginx NodeIP>:<Nginx NodePort>  # nginx HTTP response body 확인
```
Nginx가 정상적으로 실행되고 있다면 다음과 같은 결과를 확인할 수 있습니다.

```text
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Nexshop Marketing</title>
    <link href="https://fonts.googleapis.com/css?family=Open+Sans:400,600,700" rel="stylesheet">
    <!--<link rel="stylesheet" href="../material-design-lite/material.min.css">-->
    <!--<script src="../material-design-lite/material.min.js"></script>-->
    <!--<link rel="stylesheet" href="https://fonts.googleapis.com/icon?family=Material+Icons">-->
<link rel="shortcut icon" href="/favicon.png"><link href="/nexshopApp.styles.c5d4bccb9c15e8e4b03f.css" rel="stylesheet"></head>
<body>
    <div id="root"></div>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/paho-mqtt/1.0.2/mqttws31.min.js"></script>
<script type="text/javascript" src="/vendors.bundle.c5d4bccb9c15e8e4b03f.js"></script><script type="text/javascript" src="/react.bundle.c5d4bccb9c15e8e4b03f.js"></script><script type="text/javascript" src="/nexshopApp.bundle.c5d4bccb9c15e8e4b03f.js"></script></body>
```

#### Nginx 설정 파일 확인

Nginx 컨테이너에 접속해봅시다.

```bash
$ kubectl exec -it [nginx pod name] bash
$ ls -al servers
```
Nginx 프로세스는 /etc/nginx/servers 디렉토리 내 nxs-nginx.conf 설정 파일을 사용합니다.

중요한 것은 이 설정 파일에 `/api`로 시작하는 요청을 받으면 `http://nginx-ingress-controller:80`로 프록시하도록 설정되어 있습니다.

`http://nginx-ingress-controller:80`에서 실행중인 Service가 Nginx 요청을 받아 처리한다는 것을 짐작할 수 있습니다.

두번째로 확인해 볼 내용은 위에서 실행한 요청의 결과가 무엇인지 확인해보는 것입니다.

`/`로 요청한 경우 `index.html`이 반환됨을 알 수 있습니다.

`/var/www/html/index.html`을 확인해보면 위에서 확인한 HTTP Response Body와 일치함을 알 수 있습니다.

```xml
server {

  server_name _;

  index index.html;
  proxy_http_version       1.1;
  proxy_set_header         Connection "";
  proxy_connect_timeout    2;
  proxy_next_upstream      error timeout invalid_header http_500;

  location ^~ /api/ {
    client_max_body_size 500m;
    proxy_pass http://nginx-ingress-controller;
  }

  location / {
    limit_except GET HEAD POST PUT DELETE {
        deny all;
    }
    try_files $uri $uri/ @rewrites;
  }

  location @rewrites { rewrite ^(.+)$ /index.html last; }

  root /var/www/html;

}
```

### Ingress Resource
#### Ingress Resource 정의
Ingress Resource는 두가지로 나누어 구성하였습니다.
- Auth Service로부터 토큰을 발급 받고 유효한 토큰을 포함한 요청만 허용하는 경우
- 인증/권한 처리가 필요없는 요청의 경우

첫번째 Ingress Resource에는 요청을 라우팅 하기 전에 Authentication/Authorization과 같은 선행 작업이 필요한 요청들을 Path 기반으로 정의하였습니다.

`annotations: nginx.ingress.kubernetes.io/auth-url`을 이용하여 Auth Service를 지정할 수 있습니다.

이 때 Auth Service의 hostname은 `[serviceName].[service namespace].svc.cluster.local`로 지정해야 합니다.

이 Ingress Resource에 정의한 Path와 매칭되는 요청이 들어올 경우 요청을 실행하기 전에 항상 Auth Service에게 클라이언트가 보낸 토큰이 유효한지 확인합니다.

이 때 Auth Service는 HTTP response에 4가지 Header 값을 추가적으로 돌려줍니다.

이 값을 받아 사용자 요청 Header에 싣어 보내기 위해 `nginx.ingress.kubernetes.io/auth-response-headers`에 응답으로 받은 Header key를 정의합니다.

이외에 호출에 필요한 속성을 [ingress-nginx annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md)에서 찾아 사용할 수 있습니다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nxs-service-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: http://auth.default.svc.cluster.local:9000/api/v1/auth/
    nginx.ingress.kubernetes.io/auth-method: 'POST'
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    nginx.ingress.kubernetes.io/auth-response-headers: NDE-TENANT-ID, NDE-STORE-ID, NDE-USER-ID, NDE-ID
spec:
  rules:
  - http:
     paths:
     - path: /api/v1/master-menus
       backend:
         serviceName: admin
         servicePort: 5000
     - path: /api/v1/stores
       backend:
         serviceName: admin
         servicePort: 5000
     - path: /api/v1/tags
       backend:
         serviceName: admin
         servicePort: 5000
     - path: /api/v1/contents
       backend:
         serviceName: contents
         servicePort: 5000
     - path: /api/v1/folders
       backend:
         serviceName: contents
         servicePort: 5000
```

두번째 Ingress Resource는 로그인 요청과 같이 Authentication/Authorization 확인이 필요하지 않은 요청을 정의합니다.

/api/v1/signgin 요청은 발급된 토큰이나 권한없이 백엔드 서비스를 호출할 수 있어야 합니다.

첫번째 Resource와 다르게 `annotations: nginx.ingress.kubernetes.io/auth-url` 부분이 빠진 것을 확인할 수 있습니다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: omit-service-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
spec:
  rules:
  - http:
     paths:
     - path: /api/v1/signin
       backend:
         serviceName: admin
         servicePort: 5000
```

#### Ingress Resource 생성 및 확인
다음 명령을 통해 Ingress Resource 생성합니다.

```bash
$ kubectl create -f nxs-ingress-rule.yml
$ kubectl create -f nxs-ingress-omit-rule.yml
$ kubectl get ingress
```

Ingress Resource 자체만으로는 요청을 라우팅 하는 과정에서 어떠한 영향도 주지 못합니다.

Ingress Resource는 Ingress Controller에 의해 참조되어 라우팅 여부를 결정하는데 활용됩니다.

### Ingress Controller
#### Ingress Controller 정의

Ingress Resource를 이용하여 Nginx가 보낸 요청을 처리하는 Ingress Controller 인스턴스를 생성해보도록 합시다.

Ingress Controller는 Cluster 내로 들어오는 모든 서비스 클라이언트 요청을 받는 진입점 역할을 합니다.

받은 요청을 어디로 라우팅할지 결정하기 위해 Cluster에 등록된 Ingress Resource들을 확인합니다.

서비스 클라이언트가 Ingress Resource에 등록되지 않은 Path를 요청한 경우 해당 요청을 `default-backend-service`에 정의한 인스턴스로 라우팅 합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
spec:
  ports:
  - name: port-1
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-ingress-controller
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.18.0
          args:
            - /nginx-ingress-controller
            - --default-backend-service=$(POD_NAMESPACE)/default-backend-service
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          ports:
          - name: http
            containerPort: 80
```

#### Ingress Controller 생성 및 접속

다음 명령을 통해 Service를 생성합니다.

```bash
$ kubectl create -f nginx-ingress-controller.yml
```

Nginx Ingress Controller가 정상적으로 실행되고 있는지 확인해봅시다.

```bash
$ kubectl get pods  # nginx가 실행중인 pod 확인
$ kubectl get pods [nginx pod name] -o wide  # pod이 실행중인 NodeIP 확인
$ kubectl get svc nginx-ingress-controller # NodePort 확인
$ curl http://<Nginx Ingress Controller NodeIP>:<Nginx Ingress Controller NodePort>  # nginx HTTP response status code 확인
```
Nginx Ingress Controller가 정상적으로 실행되고 있다면 다음과 같은 결과를 확인할 수 있습니다.

```text
<html><body><h1>It works!</h1></body></html>
```

Nginx Ingress Controller가 정상적으로 요청을 라우팅하는지 확인해봅시다.

요청을 실행하기 전에 `nginx`, `nginx-ingress-controller`와 `admin` Pod의 실행 로그를 볼 수 있도록 준비합니다.

```bash
$ kubectl get pods  # nginx-ingress-controller, admin pod name 확인
$ kubectl logs -f [nginx-ingress-controller pod name]
$ kubectl logs -f [admin pod name]
```
브라우저나 쉘을 이용해 Nginx 서버에 로그인 API `POST /api/v1/signin`를 테스트 합니다.

HTTP Request Body는 다음과 같습니다.
```json
{ "userId":"jacky.shin",
  "password":"MXEydzNlNHI1dA=="
}
```

이 요청은 Nginx -> Nginx Ingress Controller -> Admin 순서로 실행됩니다.

Nginx에 `/api/v1/signin`를 호출하고 HTTP Response에 `Set-Cookie: NDE-AUTH-TOKEN` 을 확인합니다.

```bash
$ curl -i -XPOST -H 'Content-Type: application/json; charset=utf-8' -d '{"userId":"jacky.shin", "password":"MXEydzNlNHI1dA=="}' http://<Nginx NodeIP>:<Nginx NodePort>/api/v1/signin
```

정상적으로 요청이 실행되었다면 로그와 함께 다음과 같은 HTTP Response를 확인할 수 있습니다.

```text
HTTP/1.1 200
Server: nginx/1.15.3
Date: Fri, 28 Sep 2018 01:58:48 GMT
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
X-Application-Context: admin-service:apit:5000
Set-Cookie: NDE-AUTH-TOKEN=447cc33457bcc7f28d184a333dc9ef87; Path=/; HttpOnly
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: POST, GET, OPTIONS, DELETE
Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept, NDE-AUTH-TOKEN
X-adminApp-alert: adminApp.userInfo.created
X-adminApp-params: token
```

### Auth service
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
