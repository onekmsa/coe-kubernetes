# admin-service 배포

기존에 Netflix Stack 기반으로 Cloud 위에서 운영 되었던 서비스를  
Kubernetes(이하 K8S)에 배포하기 위한 전환 가이드입니다.

## 1. 프로젝트 코드 수정
Netflix Stack 중 K8S가 제공하는 기능으로 대체할 수 있는 라이브러리를 제거하고 관련된 코드를 수정합니다.  
### Gradle 수정  
**불필요한 Dependency 삭제 - Eureka Client, Spring Cloud Config**
```
// REMOVED LIB LIST
//  compile 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
//  compile "org.springframework.cloud:spring-cloud-starter-bus-amqp"
//  compile "org.springframework.cloud:spring-cloud-starter-turbine"
//  compile 'org.springframework.cloud:spring-cloud-starter-config'
```
- 위 라이브러리의 기능은 K8S에서는 아래 목록의 대체제를 사용한다.
# TODO
| Lib 명 | Netflix Stack | K8S |
|:---------|:---------:|:---------:|
|spring-cloud-starter-netflix-eureka-client|Service Discovery| K8S DNS |
|spring-cloud-starter-config | Property Set | K8S ConfigMap   |

### Property 수정
bootstrap.yml, application.yml등에서 아래 목록과 관련된 설정이 있다면 삭제 합니다.   

- Eureka 관련 정보
- Spring Cloud Config 설정 정보

```yaml
# src/main/resources/bootstrap.yaml

spring:
  application:
    name: admin-service

#**********************************************************************************************************
# REMOVE FOR K8S
#**********************************************************************************************************
#spring:
#  cloud:
#    config:
#      discovery:
#        enabled: true
#        serviceId: config-server
#
#eureka:
#  client:
#    serviceUrl:
#      defaultZone: ${EUREKA_SERVER_URL:http://serka01}:${EUREKA_SERVER_PORT:8761}/eureka,${EUREKA_SERVER_URL:http://serka02}:${EUREKA_SERVER_PORT:8761}/eureka
#    enabled: true
#
```

K8S에 배포된 내부 서비스들을 호출하기 위해서는 DNS에 등록된 서비스 오브젝트의 네임을 사용합니다. 기존에 등록된 호스트 정보를 K8S 서비스 명으로 변경합니다.
(만약, Eureka Instance Name과 동일한 이름으로 K8S 서비스명을 등록했다면 아래 과정은 생략 가능합니다.)
```sh
$ kubectl get service

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
admin                      NodePort    10.111.76.105    <none>        5000:31724/TCP                8d
auth                       NodePort    10.111.158.16    <none>        9000:30180/TCP                6d
contents                   NodePort    10.108.19.164    <none>        5000:32458/TCP                8d
iam                        ClusterIP   10.97.249.16     <none>        9543/TCP                      7d
kubernetes                 ClusterIP   10.96.0.1        <none>        443/TCP                       16d
mariadb                    ClusterIP   None             <none>        3306/TCP                      14d
mongo                      ClusterIP   None             <none>        27017/TCP                     9d
nginx-ingress-controller   NodePort    10.109.39.109    <none>        80:31136/TCP                  7d
nxs-ui                     NodePort    10.107.135.71    <none>        80:30088/TCP                  7d
rabbitmq                   ClusterIP   None             <none>        5672/TCP,4369/TCP,25672/TCP   13d
rabbitmq-management        NodePort    10.109.123.2     <none>        15672:30025/TCP               13d
redis-nodes                NodePort    10.106.120.24    <none>        6379:30520/TCP                15d
```

K8S 내부에 배포 된 mariaDB, redis, rabbitMQ를 K8S 서비스 명으로 호출합니다. (K8S 내부 호출이므로 Cluster Port를 사용합니다.)
```yaml
# src/main/resources/application.yaml

spring:
  datasource:
    driverClassName: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://${DB_HOST:mariadb}:${DB_PORT:3306}/${DB_SCHEMA:admin_service_apit}
    ...
  redis:
    host: ${REDIS_HOST:redis-node}
    port: ${REDIS_PORT:6379}
  rabbitmq:
    #    addresses: ${RABBITMQ_ADDR:amqp://guest:guest@rabbitmq:5672}
    host: rabbitmq
    port: 5672
    ...
```

프러퍼티를 통해 관리되는 login url을 web-ui 서비스명으로 호출합니다.   
```yaml
# src/main/resources/application.yaml

application:
  nexshop:
    login:
      url: http://nxs-ui/common/login
```

iam service 프러퍼티에서 iam 서버 호스트 정보를 서비스명으로 변경합니다.
```properties
# src/main/resources/iam.service.properties
iam.server.connection.host=iam
iam.server.connection.http.port=9543
```

*** iam 서버는 K8S 내부에서 구동되지 않지만 외부 아이피 정보를 아래와 같이 K8S 서비스 오브젝트로 등록하였습니다.  
서비스 오브젝트로 관리하면 아이피가 변경되었을 때 어플리케이션 레벨 수정없이 반영할 수 있습니다.  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: iam
spec:
  clusterIP: 10.97.249.16
  ports:
  - port: 9543
    targetPort: 9543
```


### Source Code 수정
AdminApp.java에서 Eureka Discovery 어노테이션을 삭제합니다.   

Eureka 설정은 더이상 참조하지 않으므로 Eureka Config Bean 설정 파일이 있다면 함께 삭제합니다.  
(현 프로젝트에서는 config/AwsAwareEurekaInstanceConfig.java 파일을 삭제 했습니다.)
```java
// AdminApp.java

//@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients
@EnableHystrix
@EnableConfigurationProperties( ApplicationProperties.class)
@RefreshScope
public class AdminApp {
  ...
}
```

feignClient에서 url을 통해 서비스를 직접 호출하도록 수정합니다.

feignClient는 name에 서비스명을 입력할 경우 eureka 레지스트리 정보로 타 서비스를 호출합니다.  
eureka를 더이상 사용하지 않으므로 url에 호스트 정보를 주어 직접 호출하도록 변경합니다.  
이 때, K8S 내부 어플리케이션들은 서비스 오브젝트의 name으로 통신할 수 있으므로 서비스명:ClusterPort로 직접 호출하도록 합니다.
(url 정보는 위의 Property 수정 시 msa.application.content=contents:5000로 변경함)        
```java
// apis/content/ContentClient.java
@FeignClient(
        value = "contents"
        url = "${msa.application.content}",
        decode404 = true,
        configuration = ContentClient.MultipartSupportConfig.class
)
public interface ContentClient {
  ...
}
```

feignClient를 통해 타 서비스를 호출하는 다른 파일들도 동일하게 수정합니다.
```java
// apis/device/DeviceClient.java
@FeignClient(value = "device" url = "${msa.application.device}", decode404 = true)
public interface DeviceClient {
  ...
}
```
```java
// apis/push/NotificationPushClient.java
@FeignClient(value = "push" url = "${msa.application.notificationPush}")
public interface NotificationPushClient {
  ...
}
```  


## 2. Docker - Build Image & Push
수정된 프로젝트를 빌드하여 도커 이미지로 만든 후 레지스트리에 푸시합니다.

### Dockerfile 생성
프로젝트 경로에 Dockerfile을 작성합니다.
```
FROM openjdk:8-jre-alpine
ENV APP_FILE marketing-admin-1.1.0-SNAPSHOT.jar
ENV APP_HOME /usr/app
COPY build/libs/$APP_FILE $APP_HOME/
WORKDIR $APP_HOME
ENTRYPOINT ["sh", "-c"]
CMD ["exec java -jar $APP_FILE"]
```
### Image Build
프로젝트를 빌드하여 도커파일에서 설정한 경로에 jar 파일이 만들어지면 아래 명령어로 도커 이미지를 생성합니다.
```sh
$ docker build -t docker.sds-act.com/admin .
```
### Image Push
```sh
$ docker push docker.sds-act.com/admin
```

## 3. Kubernetes Run
### Docker Registry Secret 설정
프라이빗 도커 레지스트리에서 이미지를 받아오기 위해서는 Secret설정이 필요합니다.  
아래와 같이 Secret을 생성한 후 ServiceAccount에 적용합니다. (기존에 적용되어 있다면 생략해도 됩니다.)
```
$ kubectl create secret docker-registry coe-registry-key --docker-server=https://docker.sds-act.com --docker-username=dockeruser --docker-password=yourPassword

$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "coe-registry-key"}]}'
```
### Deployment
```yaml
# admin-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin
  labels:
    app: admin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: docker.sds-act.com/admin:latest
        ports:
        - containerPort: 5000
```
### Service
```yaml
# admin-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: admin
spec:
  selector:
    app: admin
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
  type: NodePort  # K8S 외부에서 접속 확인을 위해 NodePort로 설정하였으나
                  # 내부 서비스 호출만 일어날 경우 ClusterIP로 변경합니다.
```   

```sh
$ kubectl create -f admin-deployment.yaml
$ kubectl create -f admin-svc.yaml
```

![](../images/k8s-admin-deploy.png)
