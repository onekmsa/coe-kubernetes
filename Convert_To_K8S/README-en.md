## 1. Before Start

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

## 2. Let's start

### 2.1 Client request to Gateway : Ingress
  converting Netflix OSS Zuul to K8S-Ingress....
   - [how to setup Ingress][ConvertIngress]
     - install Nginx Ingress
     - configure routing rules
     - [auth-service](./ingress/auth-service-deploy.md)

### 2.2 Why Need Auth Service?
   <img src="../image/zuul_filter.png" width="500">   

  - Zuul offers filters like the above image, but Ingress doesn't.  
  Only configuring routing rules are available on Ingress. That's why we have to run a seperate auth service.     

  - [Auth-Service](./ingress/auth-service-deploy.md)'s features are equal to Zuul Filters'
    - Authentication
    - Authorization

### 2.3 Let's Run Applications

#### 2.3.0 Remove Netflix OSS Settings (in case of NETFLIX OSS Applied Projects)
 - It's optional to use Netflix OSS stack on K8S environment. If you want to utilize K8S functions more, this guide is recommended.  
 - Key Settings
   - remove Netflix OSS configurations (Eureka, Spring Cloud Config..)
   - convert IP based requests to K8S DNS resolver based requests  
   [Converting Guide](./service_converting/contents/modify_netflix_in_content.md)    

> **How applications connect each other on K8S?**  
 Internal or External calling is available with Service Object's Service Name.
 [What's Service Object?][Service]  

#### 2.3.1 Dockerize an application
- You have to build a docker image to deploy an application on K8S
[Dockerize](./service_converting/contents/dockerize_content.md)
#### 2.3.2 Run in K8S
- Deploying images on K8S
[K8S Deploy](./service_converting/contents/run_content_in_k8s.md)
#### Appendix: Deployment with CI/CD
- [K8S Jenkins Deploy](../3.CICD/kubernetes_deploy.md)
#### Appendix: Deployment of Database on K8S
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
  [Service]: ../2.개념/kubernetes.md#L131 "Service"

  [Ingress]: ../2.%EA%B0%9C%EB%85%90/kubernetes-ingress.md "Ingress"

  [ConvertIngress]: ./ingress/README.md "ConvertIngress"

  [KubernetesService]: https://kubernetes.io/docs/concepts/services-networking/service/ "KubernetesService"
