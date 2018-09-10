# Kubernetes VS Spring cloud

## 1. 개요

MSA 환경에서 Kuberntes(이하 쿠버네티스)와 Spring Cloud(Netflix 스택 포함)는 가장 많이 사용 되는 솔루션입니다.

마이크로서비스를 개발하고 실행하기 위한 최상의 환경이지만 비슷하면서 다른 성격을 가지고 있습니다.

마이크로서비스는 이미 대중적으로 많이 알려졌고 모듈의 독립적인 배치와 기술의 다양성을 지원하지만 분산 시스템을 개발해야 하고 운영에 있어서 많은 비용이 발생 할 수 있습니다.

모듈 또는 서비스의 분리로 인해서 아래와 같은 관리 포인트가 늘어나게 됩니다.

* 설정 관리(Config Management)

* 서비스 디스커버리

* 중앙집중 로깅 (Centralized Logging)

* 분산 추적 (Distributed Tracing)

* Auto Scaling

* Security

* API Management

* Fault Tolerance

물론 위의 고려사항은 조직구조, 문화 등 비 기술적 문제는 포함되지 않습니다.

## 2. 기술 스택 비교

### 2.1 사용 기술

| Microservice                       | Spring Cloud & Netflix OSS              | Kubernetes                                          |
|------------------------------------|-----------------------------------------|-----------------------------------------------------|
| Config Management                  | Config Server, Consul, Netflix Archaius | Config Map & Secret                                 |
| Load Balancing                     | Netflix Ribbon                          | Kubernetes Service                                  |
| Service Security                   | Spring Cloud Security                   | -                                                   |
| API Gateway                        | Netflix Zuul                            | Kubernetes Service & Ingress Resources              |
| Centralized Logging                | ELK                                     | EFK                                                 |
| Centralized Metrics                | Netflix Spectator & Atlas               | Heapster, Prometheus, Grafana                       |
| Distributed Tracing                | Spring Cloud Sleuth, Zipkin             | OpenTracing, Zipkin                                 |
| Resilience & Fault Tolerance       | Netflix Hystrix, Turbine & Ribbon       | Kubernetes Health Check & Resource Isolation        |
| Auto Scaling & Self Healing        | -                                       | Kubernetes Health Check, Self Healing, Autoscailing |
| Packaging, Deployment & Scheduling | Spring Boot                             | Docker/Rkt, Kubernetes Scheduler & Deployment       |
| Job Management                     | Spring Batch                            | Kubernetes Jobs & Scheduled Jobs                    |
| Singleton Application              | Spring Cloud Cluster                    | Kubernetes Pods                                     |


Spring Cloud의 런타임 문제를 해결 할 수 있는 풍부한 Java 라이브러리가 존재합니다.
Spring Cloud 진영은 NetflixOSS의 검증된 스택들을 체택하였고 이러한 검증된 Java라이브러리를 통해서 Java가 익숙한 개발자가 마이크로서비스를 쉽게 적용 할수 있습니다.
다만 마이크로서비스 적용을 위해서는 라이브러리나 별도의 에이전트를 코드 상에 추가해야 한다는 단점이 존재합니다.

쿠버네티스의 경우 Java에 국한되지 않고 다중언어를 지원하기 위해 일반적인 방식으로 문제를 해결 합니다. 따라서 어플리케이션 자체에 특별한 라이브러리나 에이전트를 추가할 필요가 없습니다.

특정 고려사항(예를 들어 로깅)에서는 비슷하거나 동일한 기술 스택을 사용하여 해결하지만 다른 부분에서는 전혀 다른 기술 스택을 사용하여 문제를 해결하고 있습니다.

이런 경우 독립적으로 하나의 해결책만 사용할 수 있겠지만 같이 사용하여 서로를 보완 할 수 있습니다.

예를 들어 Hystrix의 경우 탄력적인 내결함성 마이크로서비스를 제공 할 수 있지만 Kubernetes의 health check, auto-scailng 등의 기능과 결합 되면 완벽한 결함 방지 시스템을 구축 할 수 있습니다.

## 3. 장 단점

### 3.1 Spring Cloud

개발자가 신속하게 설정관리, discovery service, circuit breaker를 구현 할 수 있으며 Java 개발자를 위해 Netflix OSS라이브러리 위해서 구축 됩니다.

* 장점

  * 가장 강력한 플랫폼 중 하나인 Spring이 제공하는 여러 기능 덕분에 개발자는 친숙한 환경에서 마이크로서비스를 구축 할 수 있습니다.

  * 어노테이션 하나로 간단히 마이크로서비스 스택 기능을 구현 할 수 있습니다.

  * 다양한 Spring Cloud 라이브러리들은 잘 통합 되어 있습니다. 예를 들어 feign의 경우 hystrix와 ribbon을 통하여 로드밸런싱 요청을 수행 할 수 있습니다.

* 단점

  * 제일 큰 문제점 중 하나는 Java에만 국한 된다는 점입니다. 물론 사이드카 패턴 기능을 구현하여 JVM 언어가 아닌 타 언어로 작성 된 응용 프로그램을 적용 할 수 있지만 한계가 존재합니다.

  * 기존의 소스 코드를 수정 할 곳이 많아지며 개발자에게 많은 책임이 주어진다는 단점이 있습니다. 예를 들어 ConfigServer를 사용하고 있고 개발자가 개발단계에서 있어서 하나의 마이크로 서비스를 실행하고자 하면 Config Server를 구동 시켜야 합니다.

### 3.2 쿠버네티스

쿠버네티스는 컨테이너로 만들어진 응용 프로그램들을 쉽게 배치하고 확장시키며 관리 할 수 있는 오픈소스 플랫폼입니다.

* 장점

  * 쿠버네티스는 언어에 독립적이며 다중 언어를 지원하는 컨테이너 기반의 플랫폼입니다.

  * 쿠버네티스는 리소스에 대한 제약 조건을 설정하며 RBAC를 관리하고 어플리케이션에 대한 라이프 사이클을 관리하고 auto-scailng을 가능하게 합니다.

* 단점

  * 쿠버네티스는 다중언어를 지원하기 때문에 Spring Cloud와 같은 다양한 플랫폼에 맞게 최적화가 되지 않습니다.

  * Spring Cloud의 경우 개발자 중심의 플랫폼이지만 쿠버네티스는 그렇지 않습니다. 따라서 개발자가 쿠버네티스를 사용하게 될 경우 새로운 개념을 습득 해야 합니다.

## 4. 결론

각자 다른 두 플랫폼은 비슷하면서 다른 특징을 가지고 있습니다.

그리고 개발자에게 요구하는 기술적인 스택뿐만 아니라 그것을 사용함에 있어서 필요한 조직 문화도 서로 다를 수 있습니다.

모놀로틱 레거시 서비스를 마이크로서비스로 전환함에 있어 Spring Cloud 스택만을 적용할지 쿠버네티스 스택만을 적용할지를 생각해야 합니다.

Spring Cloud 스택의 경우 MSA에 대한 적용을 JVM내부에서 해결하고자 하지만 쿠버네티스의 경우 자신의 플랫폼 수준에서 해결하려고 합니다. 따라서 2개의 융햡이 가능하기 때문에 Java 어플리케이션을 이용한다면 두 가지 모두 사용을 할 수도 있습니다.

예를 들어 Spring Application은 maven과 같은 툴을 이용하여 어플리케이션을 패키징을 하고 쿠버네티스는 배포와 스케줄링을 제공 할 수 있습니다. 그리고 Hystrix 쓰레드 풀을 통해 어플리케이션 내부에서 bulkheading을 제공하고 쿠버네티스는 리소스, 프로세스, 네임스페이스 분리를 통해서 bulkheading을 제공하기 때문에 더욱더 완성도 높은 시스템을 구축 할 수 있습니다.

### 5. 참고
https://developers.redhat.com/blog/2016/12/09/spring-cloud-for-microservices-compared-to-kubernetes/
