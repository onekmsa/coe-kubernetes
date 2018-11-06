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
- {{SERVICE_NAME}}-deployment.yaml
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: {{SERVICE_NAME}}
    labels:
      app: {{SERVICE_NAME}}
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: {{SERVICE_NAME}}
    template:
      metadata:
        labels:
          app: {{SERVICE_NAME}}
      spec:
        containers:
        - name: {{SERVICE_NAME}}
          image: docker.sds-act.com/{{SERVICE_NAME}}:latest
          ports:
          - containerPort: 8080
          env:
          - name: RABBITMQ_SERVER_URL
            value: "rabbitmq"
  ```
- {{SERVICE_NAME}}-svc.yaml
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: {{SERVICE_NAME}}
  spec:
    selector:
      app: {{SERVICE_NAME}}
    ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
    type: NodePort
  ```
