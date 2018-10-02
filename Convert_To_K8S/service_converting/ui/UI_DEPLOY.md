
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
