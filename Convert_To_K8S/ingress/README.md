# Netflix Zuul Migration to Ingress

## Table of Contents
- Kubernetes Ingress 소개
- Compare Zuul with Ingress
- Migrate Zuul to Ingress

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

|                           | 역할                                                                        |
|---------------------------|----------------------------------------------------------------------------|
| Ingress Resource          | Ingress가 어떤 요청을 어떤 어플케이션으로 라우팅할지 hostname, path 정보 등을 기반으로 매핑 |
| Ingress Controller 구현체   | Ingress Resource에 정의한 규칙을 기반으로 요청을 실행하는 인스턴스                     |                    

## 2. Compare Zuul with Ingress

### 2.1 Netflix OSS를 이용한 환경

아래 그림에서 Zuul은 Nginx로부터 사용자 요청을 받습니다.

그 다음 이 요청을 Zuul configuration에 정의된 라우팅 규칙에 따라 적절한 백엔드 서비스로 포워딩 합니다.

이 때 Eureka를 사용한다면 Zuul은 최근 갱신된 백엔드 서비스의 IP와 Port 정보를 받아볼 수 있습니다.

```
          Web Browser
              |
          [ Nginx ]
              |
       [ Zuul & Eureka ]
---------|------------|----------
[ Services - IAM, Admin, Content ]
```

예제의 Zuul 서비스가 제공하는 기능은 다음과 같습니다.

```text
a. Request Filtering & Forwarding
b. Authentication & Authorization 확인
```
### 2.2 Ingress를 이용한 환경

아래 그림과 같이 Zuul을 Ingress Resource/Controller로 전환하였습니다.

이 때 Auth 관련 비즈니스 로직은 새로운 어플리케이션으로 분리해야 합니다.

새로 생성한 Auth Service를 Ingress Resource와 연결시키면 Zuul 서비스 기능을 모두 충족시킬 수 있습니다.

```
              Web Browser
                  |
              [ Nginx ]
                  |
    [ Ingress Resource/Controller ]
---------|------------------|----------
[ Services - Auth, IAM, Admin, Content ]
```


## 3. Migrate Zuul to Ingress
<img src="../../image/Nginx_Ingress.png" width="700">
### Ingress 환경 구성 미리 보기
각 구성 요소를 생성하기 위한 관련 파일과 내용은 다음과 같습니다.

|                    |                             파일 이름                             |                                                                       설명                                                                       |
|:------------------:|-----------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
|    Ingress Resource    | ingress/nxs-ingress-rule.yaml, ingress/nxs-ingress-omit-rule.yaml | Cluster에서 처리할 Inbound traffic을 Path 기반으로 정의한 Ingress definition                                                                     |
| Ingress Controller | ingress/nginx-ingress-controller.yaml                             | Nginx를 통해 전달받은 Inbound traffic을 Ingress Resource에 따라 필터링하고 Service로 포워딩하는 작업을 담당하는 Nginx Ingress Controller definition  |
|    [Auth Service](./auth-service-deploy.md)    | auth/auth-svc.yaml, auth/auth-deployment.yaml                     | User request를 Service로 포워딩 하기 전에 실행할 Authentication/Authorization 확인 로직을 담은 백엔드 Service Definition |                                           |


### 3.1 Ingress Controller
#### 3.1.1 Ingress Controller 정의

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

#### 3.1.2 Ingress Controller 생성

다음 명령을 통해 Service를 생성합니다.

```bash
$ kubectl create -f nginx-ingress-controller.yml
```

Nginx Ingress Controller가 정상적으로 실행되고 있는지 확인해봅시다.

```bash
$ kubectl get pods  # nginx가 실행중인 pod 확인
$ kubectl get pods [nginx pod name] -o wide  # pod이 실행중인 NodeIP 확인
$ kubectl get svc nginx-ingress-controller # NodePort 확인
```

### 3.2 Ingress Resource
#### 3.2.1 Ingress Resource 설정 방법

Ingress Resource는 두가지로 나누어 구성하였습니다.

##### 3.2.1.1 Auth Service로부터 토큰을 발급 받고 유효한 토큰을 포함한 요청만 허용하는 경우

```yaml
# nxs-ingress-rule.yml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nxs-service-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: http://auth.default.svc.cluster.local:9000/api/v1/auth/    # (A)
    nginx.ingress.kubernetes.io/auth-method: 'POST'                                                   
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'                                                 
    nginx.ingress.kubernetes.io/auth-response-headers: NDE-TENANT-ID, NDE-STORE-ID, NDE-USER-ID, NDE-ID  # (B)
spec:
  rules:                                      #(C)
  - http:
     paths:
     - path: /api/v1/master-menus     # Matched Request Url
       backend:
         serviceName: admin           # Service Name
         servicePort: 5000            # Service ClusterPort
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

 - `annotations: nginx.ingress.kubernetes.io/auth-url` (A) : Ingress Resource에 정의한 Path와 매칭되는 요청이 들어올 경우 해당 서비스로 라우팅 전에 위에서 정의한 url로 리다이렉트됩니다. 이 예제의 경우 Auth-Service를 통해 사용자 인증 체크를 합니다. 이 때 주의할 점은 Auth Service의 hostname은 반드시 `[serviceName].[service namespace].svc.cluster.local`로 지정해야 합니다.

 - `nginx.ingress.kubernetes.io/auth-response-headers` (B) : 권한 서비스에서 받은 Response-Header값을 사용자 Request Header에 함께 실어 보내기 위해서 Header Key값을 지정합니다.
 - spec.rules(C)에 라우팅 룰을 정의합니다.

##### 3.2.1.2 인증/권한 처리가 필요없는 요청의 경우

두번째 Ingress Resource는 로그인 요청과 같이 Authentication/Authorization 확인이 필요하지 않은 요청을 정의합니다.

```yaml
# nxs-ingress-omit-rule.yml
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

현재 예제에서 /api/v1/signgin 요청은 발급된 토큰이나 권한없이 백엔드 서비스를 호출할 수 있어야 합니다.
위 Resource와 다르게 `annotations: nginx.ingress.kubernetes.io/auth-url` 부분이 빠진 것을 확인할 수 있습니다.


> 이외에 호출에 필요한 속성은 [ingress-nginx annotations](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md) 이 링크를 통해 확인 하실 수 있습니다.




#### 3.2.2 Ingress Resource 생성
다음 명령을 통해 Ingress Resource 생성합니다.

```bash
$ kubectl create -f nxs-ingress-rule.yml
$ kubectl create -f nxs-ingress-omit-rule.yml
$ kubectl get ingress
```
Ingress Resource는 Ingress Controller에 참조되어 라우팅됩니다.

## 4. 확인

Nginx Ingress Controller가 정상적으로 실행되고 있다면 다음과 같은 결과를 확인할 수 있습니다.
```bash
# nginx HTTP response status code 확인
$ curl http://<Nginx Ingress Controller NodeIP>:<Nginx Ingress Controller NodePort>  
```
```html
<html><body><h1>It works!</h1></body></html>
```

### 4.1 라우팅 실행 확인
Nginx Ingress Controller가 정상적으로 요청을 라우팅하는지 확인해봅시다.

요청을 실행하기 전에 `nginx`, `nginx-ingress-controller`와 `admin` Pod의 실행 로그를 볼 수 있도록 준비합니다.

```bash
$ kubectl get pods  # nginx-ingress-controller, admin pod name 확인
$ kubectl logs -f [nginx-ingress-controller pod name]
$ kubectl logs -f [admin pod name]
```

### 4.2 Auth Url 실행 확인  
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
