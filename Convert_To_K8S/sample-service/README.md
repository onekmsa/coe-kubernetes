# Sample-service Convert To K8S

## 1. Containerize services
  - Repository
    - [Order-Service](https://github.com/SDSACT/coe-sample-service/tree/k8s/order-service)
    - [Customer-Service](https://github.com/SDSACT/coe-sample-service/tree/k8s/customer-service)
    - [Email-Service](https://github.com/SDSACT/coe-sample-service/tree/k8s/email-service)
  - Docker Build & Push
    1. Login to Repository
    ```sh
    $ docker login {{REPOSITORY_URL}} -u {{USERNAME}} -p {{PASSWORD}}
    WARNING! Using --password via the CLI is insecure. Use --password-stdin.
    Login Succeeded
    ```
    2. Build Docker Images
    ```sh
    $ docker build -t {{SERVICE_NAME}}
    ```
    3. Tag Docker Images
    ```sh
    $ docker tag {{SERVICE_NAME}} {{REPOSITORY_URL:PORT}}/{{SERVICE_NAME}}
    ```
    4. Push Docker Repository
    ```sh
    $ docker push {{REPOSITORY_URL:PORT}}/{{SERVICE_NAME}}
    ```

## 2. Deployment Rabbitmq
- [RabbitMQ 인스턴스 올리기](https://github.com/SDSACT/coe-kubernetes/blob/master/Convert_To_K8S/rabbitmq/README.md)

## 3. Deployment services
- [Kubernetes Deploy](https://github.com/SDSACT/coe-kubernetes/blob/master/Convert_To_K8S/service_converting/contents/run_content_in_k8s.md)
- 
