# 1. Docker registry 생성 및 image push
#### docker registry 시작
```sh
$ docker pull registry
$ docker run -dit --name docker-registry -p 5000:5000 registry
```
#### 이미지생성
jar를 포함하도록 docker 이미지 생성  

Dockerfile 예시  
```Dockerfile
FROM openjdk:8-jre-alpine
ENV APP_FILE customer-service-1.0.0-SNAPSHOT.jar
ENV APP_HOME /usr/app
EXPOSE 8091
COPY target/$APP_FILE $APP_HOME/
WORKDIR $APP_HOME
ENTRYPOINT ["sh", "-c"]
CMD ["exec java -jar $APP_FILE"]
```

```sh
$ docker build -t localhost:5000/customer-service:1.0 .
$ docker images
```

#### 이미지 push
```sh
$ docker push localhost:5000/hello-world
$ curl -X GET http://localhost:5000/v2/_catalog
$ curl -X GET http://localhost:5000/v2/customer-service/tags/list
```

# 3. kubernetes에 배포
Deployment.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: customer-service
  labels:
    app: customer-service
spec:
  type: NodePort
  ports:
  - port: 8091
    name: http
  selector:
    app: customer-service
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: customer-service
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: customer-service
        version: v1
    spec:
      containers:
      - name: customer-service
        image: localhost:5000/customer-service:0.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8091
```
kubernetes에 배포하면서 동시에 proxy 추가
```sh
kubectl create -f<(istioctl kube-inject -f Deployment.yaml)
```
> 아래 명령어로 자동 inject가능  
> kubectl label namespace <namespace> istio-injection=enabled  

> docker registry 인증서를 만들어 kube에 적용했지만 localhost에서만 찾아서 안됨..  
> kubectl create secret docker-registry regcred --docker-server=서버ip:5000 --docker-username=admin --docker-password=admin --docker-email=test@mail.com

//TODO:
kube에서 localhost:5000 에서 이미지 다운로드 시도 함  
자신의 호스트 pc의 ip를 주려 했으나, https 접속해야 한다고 해서..
registry를 https로 돌리거나, 인증 무시해서 돌리는것으로 다시 테스트 필요.

쿠버내에 registry를 올려서 사용하는 것도 확인 필요  

## 참조
[Docker registry 생성 및 HTTPS 적용]   https://novemberde.github.io/2017/04/09/Docker_Registry_0.html    
[Sample app으로 쿠버, istio 적용]  https://piotrminkowski.wordpress.com/2018/04/13/service-mesh-with-istio-on-kubernetes-in-5-steps/   
