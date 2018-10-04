## 1. 시작하기 전에

### 1.1 Compare Netflix OSS with Kubernetes

| Microservice    | Spring Cloud & Netflix OSS  |Kubernetes      |
|------|------|------|
| Config Management|[Config Server][Config Server], Consul, Netflix Archaius | Config Map & Secret |
| Service Discovery                  | [Eureka][Eureka] | Kubernetes DNS                     |
| Load Balancing                     | [Netflix Ribbon][Ribbon]                | Kubernetes Service                                  |
| API Gateway                        | [Netflix Zuul][Zuul]                    | Kubernetes Service & [Ingress Resources][Ingress]   |
| Centralized Logging                | [EFK][EFK]                              | EFK                                                 |
| Centralized Metrics                | Netflix Spectator & Atlas, [SpringBootAdmin][SpringBootAdmin]| Heapster, Prometheus, Grafana  |
| Distributed Tracing                | [Spring Cloud Sleuth][Sleuth], Zipkin, [Pinpoint][Pinpoint]   | OpenTracing, Zipkin           |
| Resilience & Fault Tolerance       | [Netflix Hystrix][Hystrix], Turbine & Ribbon | Kubernetes Health Check & Resource Isolation   |
| Auto Scaling & Self Healing        | -                                       | Kubernetes Health Check, Self Healing, Autoscailing |
| Packaging, Deployment & Scheduling | Spring Boot                             | Docker/Rkt, Kubernetes Scheduler & [Deployment][Deployment]|
| Job Management                     | Spring Batch                            | Kubernetes Jobs & Scheduled Jobs                    |
| Singleton Application              | Spring Cloud Cluster                    | Kubernetes Pods                                     |




### 1.2 MicroService Architecture
<img src="../image/msa_with_netflix.png" width="600">


### 1.3 K8S Architecture
<img src="../image/msa_with_kubernetes.png" width="600">

## 2. 전환을 시작합니다.

### 2.1 요청할 때, 처음 만나는 관문 Ingress
  Netflix_OSS-Zuul을 K8S-Ingress로 ....
   - [Ingress 설정 방법][ConvertIngress]
     - Nginx Ingress 설치방법
     - Ingress 라우팅 설정
     - [인증서비스](./ingress/auth-service-deploy.md)
### 2.2 Auth Service가 왜 새로 생기나요?
   <img src="../image/zuul_filter.png" width="500">  

  - Zuul의 경우 위 그림과 같이 Filter 기능을 제공합니다.
  그러나 Ingress의 경우 Routing Rule만 제공할 뿐 Filter와 같은 기능은 제공하지 않고 있기 때문에, 별도의 Service로 해당 기능을 구현할 필요가 있습니다.

  - [Auth-Service](./ingress/auth-service-deploy.md)에서 하는 기능 = Zuul Filter에서 하던 기능
    - 권한
    - 인증

### 2.3 Service를 띄워보자
#### 잠깐, K8S 어플리케이션들은 어떻게 통신할까요?
  - K8S 내부 서비스 간 Internal 호출

    K8S의 각 Pod들은 자신의 ClusterIP를 할당받습니다.
    ```sh
    $ kubectl get pod -o wide

    NAME                                        READY     STATUS    RESTARTS   AGE       IP        
    admin-55864f5cdc-mlzwg                      1/1       Running   0          14d       10.42.0.4
    auth-76986d666f-blzb9                       1/1       Running   0          12d       10.44.0.5
    contents-5bc8bdb9d4-r4hfn                   1/1       Running   0          14d       10.42.0.3
    ```

    각 Pod 내부에서 IP로 직접 호출하거나 DNS에 등록된 호스트명({pod-ip-address}.{namespace명}.pod.cluster.local)으로 호출할 수 있습니다.
    하지만 Pod이 새로 생성될 경우에 IP는 계속 바뀌기 때문에 Pod를 직접 호출 하는건 좋은 방법이 아닙니다.

    그래서 서비스 오브젝트를 만들어 IP가 아닌 서비스명({svc명}[.{namespace명}.svc.cluster.local], []는 생략가능 )으로 호출합니다.
    자세한 내용은 [KubernetesService][KubernetesService] 참고하시기 바랍니다.

    ```sh
    $ kubectl get svc

    NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        
    admin                      NodePort    10.111.76.105    <none>        5000:31724/TCP
    auth                       NodePort    10.111.158.16    <none>        9000:30180/TCP
    contents                   NodePort    10.108.19.164    <none>        5000:32458/TCP
    ```    

  - K8S 내부 서비스에서 External 서버 호출
    외부 호출도 마찬가지로 직접 코드 내에서 외부 IP를 주어 호출할 수 있습니다.
    하지만 IP가 변경되면 코드를 변경해서 Pod을 다시 띄워줘야 하기 때문에 외부 IP를 아래와 같이 서비스 오브젝트로 관리하도록 권장하고 있습니다.
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: iam
    spec:
      clusterIP: X.X.X.X
      ports:
      - port: 9543
        targetPort: 9543
    ```
    이제 외부 서버도 Internal 호출처럼 서비스명으로 호출할 수 있습니다.


#### 2.3.0 Remove Netflix OSS Settings - NETFLIX OSS를 적용했던 프로젝트만 보세요
 - K8S 전환에 있어서 기존에 Netflix OSS Stack를 활용하는 것은 Optional 부분입니다. 조금 더 K8S의 기능을 활용하고자 한다면 이 가이드를 수행하는 것을 권장합니다.
 - 주요 변경사항
   - Eureka, Spring Cloud Config 설정제거
   - Service 호출시, Eureka를 참조하거나, IP BASE 호출을 Kubernetes DNS Name으로 변경
 - [이제 시작해보자](./service_converting/contents/modify_netflix_in_content.md)  

#### 2.3.1 Dockerize an application
- K8S에 어플리케이션을 배포하기 위해서는 먼저 도커 이미지를 생성해야 합니다.
[Dockerize](./service_converting/contents/dockerize_content.md)
#### 2.3.2 Run in K8S
- 만들어진 이미지로 K8S에 배포해 보겠습니다.
[K8S Deploy](./service_converting/contents/run_content_in_k8s.md)
#### 부록: CI/CD로도 배포해보자
- [K8S Jenkins Deploy](../3.CICD/kubernetes_deploy.md)
#### 부록: K8S에 Database를 배포해보자
- [mariadb](./mariadb/README.md)
- [mongo](./mongo/README.md)
- [redis](./redis/README.md)
- [rabbitmq](./rabbitmq/README.md)





  [Config Server]: https://coe.gitbook.io/guide/config/springcloudconfig "Config Server"
  [Eureka]: https://coe.gitbook.io/guide/service-discovery/eureka "Eureka"
  [Ribbon]: https://coe.gitbook.io/guide/load-balancing/ribbon "Ribbon"
  [Zuul]: https://coe.gitbook.io/guide/gateway/zuul "Zuul"
  [EFK]: https://coe.gitbook.io/guide/log/efk "EFK"
  [SpringBootAdmin]: https://coe.gitbook.io/guide/monitoring/spring-boot-admin "SpringBootAdmin"
  [Sleuth]: https://coe.gitbook.io/guide/log/sleuth "Sleuth"
  [Pinpoint]: https://coe.gitbook.io/guide/tracing/pinpoint "Pinpoint"
  [Hystrix]: https://coe.gitbook.io/guide/circuit-breaker/hystrix "Hystrix"
  [Deployment]: ../2.개념/deploymentstrategies.md "Deployment"

  [Ingress]: ../2.%EA%B0%9C%EB%85%90/kubernetes-ingress.md "Ingress"

  [ConvertIngress]: ./ingress/README.md "ConvertIngress"

  [KubernetesService]: https://kubernetes.io/docs/concepts/services-networking/service/ "KubernetesService"
