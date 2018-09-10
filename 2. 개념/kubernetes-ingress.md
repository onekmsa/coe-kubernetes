# Ingress

![](../image/Ingress.png)

Ingress는 쿠버네티스 내부의 서비스에 대한 요청을 처리하는 규칙의 모음입니다.

이를 사용하기 위해서는 다른 리소스와 마찬가지로 Ingress 리소스를 Kubernetes API server에 요청해야 하고 Ingress Controller가 Ingress를 수행합니다.  

일반적으로 Ingress Controller는 로드 밸런서를 이용하여 Ingress를 수행하지만 트래픽을 HA 방식으로 처리하도록 엣지 라우터 또는 프런트 엔드를 구성 할 수 있습니다.

GCE/Google Kubernetes Engine은 Ingress controller가 마스터 노드에 배포되어 있고 커스텀 Ingress Controller를 적용할 수 있으나 현재 Controller는 배타버전이므로 기능이 제한되어 있습니다. 그리고 GCE/Google Kuberentes Engine이 아닌 다른 환경이라면 Controller를 따로 배포해 주어야 합니다.  

## 1. Ingress Resource

Ingress Resource는 다음과 같은 구조로 구성 됩니다.

~~~yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
# 로드밸런서 또는 프록시 서버를 구성하는데 필요한 정보를 아래에 기재 합니다.
spec:
  rules:
  # 현재는 http규칙만 지원합니다.
  - http:
    # http 규칙에는 host, 경로 목록에 대한 정보가 포함 되어 있습니다.
      paths:
      - path: /testpath
        backend:
          serviceName: test
          servicePort: 80
~~~

## 2. Ingress Controller

Ingress 리소스가 작동을 하기 위해서는 Ingress Controller가 클러스터 내에서 동작 하고 있어야 합니다.

일반적으로 Kubernetes 구성시 자동으로 실행 되는 다른 Controller와 달리 현재 구성중인 클러스터에 가장 적합한 Ingress Controller를 선택하거나 구현 해야 합니다.

현재 쿠버네티스는 GCE 및 nginx 컨트롤러를 지원하고 있습니다.

F5 Networks는 쿠버네티스용 F5 BIG-IP 컨트롤러를 지원합니다.

Kong은 쿠버네티스에 대한 Kong Ingress Controller에 대해 지원합니다.

NGINX는 쿠버네티스를 위한 Nginx Ingress Controller를 지원합니다.


## 3. Ingress 종류

### 3.1 Single Service Ingress

일반적으로 단일 서비스의 경우 NodePort 와 같은 타입을 이용하여 서비스를 노출 할 수 있지만 이러한 규칙 없이 Ingress를 통해 접속 할 수 있습니다.

단일 서비스(nginx)를 실행합니다.
~~~bash
$ kubectl run single-service-nginx --image=nginx --port=80
$ kubectl expose deploy/single-service-nginx
$ kubectl get svc/single-service-nginx
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
single-service-nginx   ClusterIP   10.109.165.4   <none>        80/TCP    19s
~~~

현재 해당 서비스는 기본 타입인 ClusterIP이기 때문에 외부로 노출 되고 있지 않습니다.

이를 Ingress를 통해서 접속 하도록 하겠습니다.

Ingress는 하나의 규칙이기 때문에 Ingress를 사용하기 위하여 Ingress Controller를 설치합니다.

nginx ingress contoller의 경우

kubernetes/ingress-nginx, nginxinc/kubernetes-ingress with NGINX, nginxinc/kubernetes-ingress with NGINX Plus 가 존재합니다.

각 이미지 별로 설정 방법 등이 다르기 때문에 https://github.com/nginxinc/kubernetes-ingress/blob/master/docs/nginx-ingress-controllers.md 에서 확인을 하고 필요한 이미지를 이용해 컨트롤러를 기동하도록 합니다.

kubernetes/ingress-nginx 이미지를 사용하기 위해서는 default-backend 서비스가 있어야 하기 때문에 default backend 서비스도 작성하도록 합니다.

아래의 yaml을 만들어서 kubectl create -f [파일명].yaml 을 실행합니다.

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: default-backend-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: default-backend
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-backend
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: default-backend
    spec:
      containers:
      - name: default-backend
        image: httpd
        ports:
        - containerPort: 80
~~~

다음으로 ingress contoller를 생성합니다.

~~~yaml
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
          # - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --default-backend-service=$(POD_NAMESPACE)/default-backend-service
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
          - name: http
            containerPort: 80
~~~

참고) Ingress에 annotation으로 nginx-ingress 설정값을 주었습니다.
만약에 ConfigMap에서 설정할 경우 아래와 같이 ConfigMap을 생성한 후   
*** - --configmap=$(POD_NAMESPACE)/nginx-configuration *** 주석을 해제 합니다.  

~~~yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
data:
  ssl-redirect: 'false'
  rewrite-target: '/'
~~~


/foo URL로 접속할 경우 single-service-nginx로 라우팅되도록 룰을 설정 합니다.

~~~yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: single-service-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
     paths:
     - path: /foo
       backend:
         serviceName: single-service-nginx
         servicePort: 80
~~~


[Kubernetes IP]:[nginx-ingress-controller NodePort]/foo 로 접속하여 라우팅이 적용되는지 확인합니다.


참고  
http://bryan.wiki/288  
https://kubernetes.io/docs/concepts/services-networking/ingress/

### Annotation 설정

#### configuration-snippet
response header에 추가 정보를 포함합니다.

ingress에 아래와 같이 annotation을 추가합니다.
~~~yaml
nginx.ingress.kubernetes.io/configuration-snippet: |
  more_set_headers "Request-Id: $req_id";
  more_set_headers "MyHeader: myheader";
~~~

아래와 같이 nginx config 파일에 추가한 헤더 정보가 포함됩니다.   
url 호출 시 response header에 헤더 값이 전달되는 것을 확인할 수 있습니다.

~~~sh
$ kubectl exec -it [nginx-ingress-controller-pod-name] -- cat /etc/nginx/nginx.conf
~~~

~~~
location ~* ^/foo\/?(?<baseuri>.*) {
...
        more_set_headers "Request-Id: $req_id";
        more_set_headers "MyHeader: myheader";
...
~~~

#### Authentication
ingress를 통한 요청에 대해 인증서버로의 리다이렉션이 필요한 경우 인증서버 url을 지정해 줄 수 있습니다.

~~~yaml
nginx.ingress.kubernetes.io/auth-url: "http://auth-url"
~~~

인증서버에서 response header에 특정 값을 추가해준 경우  
라우팅되는 백엔드 서버 호출의 request header에 해당 값이 자동으로 포함될 수 있도록 합니다.
~~~yaml
nginx.ingress.kubernetes.io/auth-response-headers: UserID, UserGroup
~~~
