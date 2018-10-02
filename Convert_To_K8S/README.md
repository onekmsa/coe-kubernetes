## 1. 시작하기 전에

### 1.1 Compare Netflix OSS with Kubernetes

| Microservice    | Spring Cloud & Netflix OSS  |Kubernetes      |
|------------------------------------|-----------------------------------------|-----------------------------------------------------|-------------------------------|
| Config Management                  | [Config Server][Config Server], Consul, Netflix Archaius | Config Map & Secret                     |
| Service Discovery                  | [Eureka][Eureka] | Kubernetes DNS                     |
| Load Balancing                     | [Netflix Ribbon][Ribbon]                | Kubernetes Service                                  |
| API Gateway                        | [Netflix Zuul][Zuul]                    | Kubernetes Service & [Ingress Resources][Ingress]   |
| Centralized Logging                | [EFK][EFK]                              | EFK                                                 |
| Centralized Metrics                | Netflix Spectator & Atlas, [SpringBootAdmin][SpringBootAdmin]| Heapster, Prometheus, Grafana  |
| Distributed Tracing                | [Spring Cloud Sleuth][Sleuth], Zipkin, [Pinpoint][Pinpoint]   | OpenTracing, Zipkin           |
| Resilience & Fault Tolerance       | [Netflix Hystrix][Hystrix], Turbine & Ribbon | Kubernetes Health Check & Resource Isolation   |
| Auto Scaling & Self Healing        | -                                       | Kubernetes Health Check, Self Healing, Autoscailing |
| Packaging, Deployment & Scheduling | Spring Boot                             | Docker/Rkt, Kubernetes Scheduler & Deployment       |
| Job Management                     | Spring Batch                            | Kubernetes Jobs & Scheduled Jobs                    |
| Singleton Application              | Spring Cloud Cluster                    | Kubernetes Pods                                     |




### 1.2 MicroService Architecture
<img src="../image/msa_with_netflix.png" width="600">


### 1.3 K8S Architecture
<img src="../image/msa_with_kubernetes.png" width="600">

## 2. 전환을 시작합니다.

### 2.1 요청할 때, 처음 만나는 관문 Ingress
  (Netflix OSS)Zuul을 (K8S)Ingress로 ....
   - [Ingress 설정 방법][ConvertIngress]
     - Nginx Ingress 설치방법
     - Ingress 라우팅 설정
     - 인증서비
### 2.2 Auth Service가 왜 새로 생기나요?
   <img src="../image/zuul_filter.png" width="500">  

  - Zuul의 경우 위 그림과 같이 Filter 기능을 제공한다.
  그러나 Ingress의 경우 Routing Rule만 제공할 뿐 Filter와 같은 기능은 제공하지 않고 있기 때문에, 별도의 Service로 해당 기능을 구현할 필요가 있다.

  - [Auth-Service](./ingress/auth-service-deploy.md)에서 하는 기능 = Zuul Filter에서 하던 기능
    - 권한
    - 인증

### 2.2 Ingress에서 Service로
  . Netflix OSS Stack이 있었던 프로젝트라면, 제거하고 가자
  . 컨테이너화 하면 더 편해요
  . CI/CD로 자동 배포 하자
  . K8S에서 띄워 보자



  [Config Server]: https://coe.gitbook.io/guide/config/springcloudconfig "Config Server"
  [Eureka]: https://coe.gitbook.io/guide/service-discovery/eureka "Eureka"
  [Ribbon]: https://coe.gitbook.io/guide/load-balancing/ribbon "Ribbon"
  [Zuul]: https://coe.gitbook.io/guide/gateway/zuul "Zuul"
  [EFK]: https://coe.gitbook.io/guide/log/efk "EFK"
  [SpringBootAdmin]: https://coe.gitbook.io/guide/monitoring/spring-boot-admin "SpringBootAdmin"
  [Sleuth]: https://coe.gitbook.io/guide/log/sleuth "Sleuth"
  [Pinpoint]: https://coe.gitbook.io/guide/tracing/pinpoint "Pinpoint"
  [Hystrix]: https://coe.gitbook.io/guide/circuit-breaker/hystrix "Hystrix"

  [Ingress]: ../2.%EA%B0%9C%EB%85%90/kubernetes-ingress.md "Ingress"

  [ConvertIngress]: ./ingress/README.md "ConvertIngress"
