# contents-service 배포

기존에 Netflix Stack 기반으로 Cloud 위에서 운영 되었던 서비스를 Kubernetes(이하 K8S)에 배포하기 위한 전환 가이드입니다.

## 1. 프로젝트 코드 수정
Netflix Stack 중 K8S가 제공하는 기능으로 대체할 수 있는 라이브러리를 제거하고 관련된 코드를 수정합니다.  
### Gradle 수정  
**불필요한 Dependency 삭제 - Eureka Client, Spring Cloud Config**
```
// REMOVED LIB LIST
//  compile 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
//  compile 'org.springframework.cloud:spring-cloud-starter-config'
```
- 위 라이브러리의 기능은 K8S에서는 아래 목록의 대체제를 사용합니다.

| Netflix OSS Lib | Role in MSA | Substitute in K8S|
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
    name: contents-service

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

K8S에 배포된 내부 서비스들을 호출하기 위해서는 K8S DNS에 등록된 서비스 오브젝트의 네임을 사용합니다.  
기존에 등록된 호스트 정보를 K8S 서비스 명으로 변경합니다.
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

K8S 내부에 배포 된 mongoDB를 K8S 서비스 명으로 호출합니다.  
```yaml
# src/main/resources/application.yaml

spring:
  data:
    mongodb:
      uri: mongodb://admin:password@mongo/contents-service
```

### Source Code 수정
**ContentsApplication.java에서 Eureka Discovery 어노테이션을 삭제합니다.**   

Eureka 설정은 더이상 참조하지 않으므로 Eureka Config Bean 설정 파일이 있다면 함께 삭제합니다.  
(현 프로젝트에서는 config/AwsAwareEurekaInstanceConfig.java 파일을 삭제 했습니다.)
```java
// ContentsApplication.java

@EnableAspectJAutoProxy
@ComponentScan
@SpringBootApplication
@EnableHystrix
@EnableFeignClients
public class ContentsApplication {

    public static void main(String[] args) {
        SpringApplication.run(ContentsApplication.class, args);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public static Request.Options requestOptions(ConfigurableEnvironment env) {
        int ribbonReadTimeout = env.getProperty("ribbon.ReadTimeout", int.class, 70000);
        int ribbonConnectionTimeout = env.getProperty("ribbon.ConnectTimeout", int.class, 60000);

        return new Request.Options(ribbonConnectionTimeout, ribbonReadTimeout);
    }

    //////////////////////////////////////////////////////////////////////////////////
    //      REMOVE FOR K8S
    //////////////////////////////////////////////////////////////////////////////////
    //@Configuration
    //@EnableDiscoveryClient
    //public class EurekaClientConfiguration {
    //}
}
```
**Eureka 기반으로 어플리케이션을 호출하고 있는 feignClient 코드를 수정합니다.**   

feign은 다음 두 가지 방법으로 타 서비스를 호출할 수 있습니다.
  - name : eureka에 등록된 서비스 명으로 호출 (소스 코드 수정 불필요, 프러퍼티 추가 필요)
  - url : 명시된 url을 통해 직접 호출 (소스 코드 수정 필요, 프러퍼티 추가 불필요)  

name을 통해 호출할 경우 eureka 레지스트리에 등록된 서비스 명을 통해 서비스를 호출합니다.  
기존에 사용하는 Source code를 그대로 사용할 수 있는 장점이 있으나, K8S에서는 eureka 서버를 사용하지 않으므로 프러퍼티에 추가 옵션을 명시해 주어야 합니다.  

```java
// api/AdminClient.java 변경 없음
@FeignClient(name = "${msa.application.admin}", fallbackFactory = AdminClient.AdminClientFallbackFactory.class)
public interface AdminClient {
   ...
}
```

ribbon이 eureka가 아닌 프러퍼티에 명시한 서버 리스트를 통해 호출할 수 있도록 아래와 같이 추가합니다.  
이 때, K8S 내부 어플리케이션들은 서비스 오브젝트의 name으로 통신할 수 있으므로 [ServiceName]:[ClusterPort]를 입력합니다.
```yaml
# src/main/resources/application.yaml
msa:
  application:
    content: adminService

admin-service:
  ribbon:
    eureka:
      enabled: false
    listOfServers: admin:5000
```


feignClient에서 url을 직접 명시해서 호출하려면 아래 방법으로 변경합니다.  
```java
// api/AdminClient.java
@FeignClient(value = "${msa.application.admin}", url = "${msa.application.admin}", fallbackFactory = AdminClient.AdminClientFallbackFactory.class)
public interface AdminClient {
   ...
}
```

```yaml
# src/main/resources/application.yaml
msa:
  application:
    admin: admin:5000
```

feignClient를 통해 타 서비스를 호출하는 다른 파일들도 위와 동일한 방법으로 수정합니다.  


## 2. Docker - Build Image & Push
수정된 프로젝트를 빌드하여 도커 이미지로 만든 후 레지스트리에 푸시합니다.

### Dockerfile 생성
프로젝트 경로에 Dockerfile을 작성합니다.
```
FROM openjdk:8-jre-alpine
ENV APP_FILE content-service-1.1.0-SNAPSHOT.jar
ENV APP_HOME /usr/app
COPY build/libs/$APP_FILE $APP_HOME/
WORKDIR $APP_HOME
ENTRYPOINT ["sh", "-c"]
CMD ["exec java -jar $APP_FILE"]
```
### Image Build
프로젝트를 빌드하여 도커파일에서 설정한 경로에 jar 파일이 만들어지면 아래 명령어로 도커 이미지를 생성합니다.
```sh
$ docker build -t docker.sds-act.com/contents .
```
### Image Push
```sh
$ docker push docker.sds-act.com/contents
```

## 3. Kubernetes Run
#### Docker Registry Secret 설정
프라이빗 도커 레지스트리에서 이미지를 받아오기 위해서는 Secret설정이 필요합니다.  
아래와 같이 Secret을 생성한 후 ServiceAccount에 적용합니다.
```
$ kubectl create secret docker-registry coe-registry-key --docker-server=https://docker.sds-act.com --docker-username=dockeruser --docker-password=yourPassword

$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "coe-registry-key"}]}'
```
### Deployment
```yaml
# contents-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contents
  labels:
    app: contents
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contents
  template:
    metadata:
      labels:
        app: contents
    spec:
      containers:
      - name: contents
        image: docker.sds-act.com/contents:latest
        ports:
        - containerPort: 5000
```
### Service
```yaml
# contents-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: contents
spec:
  selector:
    app: contents
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
  type: NodePort  # K8S 외부에서 접속 확인을 위해 NodePort로 설정하였으나
                  # 내부 서비스 호출만 일어날 경우 ClusterIP로 변경합니다.
```   

```sh
$ kubectl create -f contents-deployment.yaml
$ kubectl create -f contents-svc.yaml
```

![](../images/k8s-contents-deploy.png)
