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

## 2. Create namesapce

```sh
$ create namespace sample-services
```

## 3. Deployment Rabbitmq
