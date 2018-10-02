# 시작하기 전에

| Microservice    | Spring Cloud & Netflix OSS  |Kubernetes      |
|------------------------------------|-----------------------------------------|-----------------------------------------------------|
| Config Management                  | [Config Server][48b2c0ee], Consul, Netflix Archaius | Config Map & Secret                     |
| Load Balancing                     | [Netflix Ribbon][Ribbon]                | Kubernetes Service                                  |
| API Gateway                        | [Netflix Zuul][Zuul]                    | Kubernetes Service & Ingress Resources              |
| Centralized Logging                | [EFK][EFK]                              | EFK                                                 |
| Centralized Metrics                | Netflix Spectator & Atlas, [SpringBootAdmin][SpringBootAdmin]| Heapster, Prometheus, Grafana  |
| Distributed Tracing                | [Spring Cloud Sleuth][Sleuth], Zipkin, [Pinpoint][Pinpoint]   | OpenTracing, Zipkin           |
| Resilience & Fault Tolerance       | [Netflix Hystrix][Hystrix], Turbine & Ribbon | Kubernetes Health Check & Resource Isolation   |
| Auto Scaling & Self Healing        | -                                       | Kubernetes Health Check, Self Healing, Autoscailing |
| Packaging, Deployment & Scheduling | Spring Boot                             | Docker/Rkt, Kubernetes Scheduler & Deployment       |
| Job Management                     | Spring Batch                            | Kubernetes Jobs & Scheduled Jobs                    |
| Singleton Application              | Spring Cloud Cluster                    | Kubernetes Pods                                     |

[48b2c0ee]: https://coe.gitbook.io/guide/config/springcloudconfig "Config Server"
[Ribbon]: https://coe.gitbook.io/guide/load-balancing/ribbon "Ribbon"
[Zuul]: https://coe.gitbook.io/guide/gateway/zuul "Zuul"
[EFK]: https://coe.gitbook.io/guide/log/efk "EFK"
[SpringBootAdmin]: https://coe.gitbook.io/guide/monitoring/spring-boot-admin "SpringBootAdmin"
[Sleuth]: https://coe.gitbook.io/guide/log/sleuth "Sleuth"
[Pinpoint]: https://coe.gitbook.io/guide/tracing/pinpoint "Pinpoint"
[Hystrix]: https://coe.gitbook.io/guide/circuit-breaker/hystrix "Hystrix"
