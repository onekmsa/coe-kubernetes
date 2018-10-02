# RabbitMQ 인스턴스 올리기

RabbitMQ 단일 인스턴스를 StatefulSet을 이용하여 올리도록 하겠습니다.

쿠버네티스 환경(minikube, kubeadm 등)이 구축 되었다고 가정하고 진행하도록 하겠습니다.

# 1. Secret 만들기

- rabbitmq-secret.yml

본 파일은 RabbitMQ pod를 만들때 필요한 환경 변수를 설정하고 있습니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-config
data:
  erlang-cookie: Yy1pcy1mb3ItY29va2llLXRoYXRzLWdvb2QtZW5vdWdoLWZvci1tZQ==
```

위와 같은 내용으로 파일을 만들고 아래의 명령으로 실행하여 Secret 리소스를 생성합니다.

~~~bash
$ kubectl create -f [파일명].yml
~~~

또는 다음과 같은 명령어로 secret을 생성할 수 있습니다.

~~~bash
$ kubectl create secret generic rabbitmq-config --from-literal=erlang-cookie=c-is-for-cookie-thats-good-enough-for-me
~~~

# 2. PersistenceVolume 만들기
데이터를 유지하기 위해서 PersistentVolume 리소스를 아래와 같이 생성합니다.

해당 예제는 볼륨을 local에 지정하였습니다.

만약 aws, glusterfs 등을 이용하여 다이나믹 프로비져닝을 할 경우에는 쿠버네티스 홈페이지를 참조하십시오. ![바로가기](https://kubernetes.io/docs/concepts/storage/volumes/)

- rabbitmq-pv.yml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
 name: pv-local-rabbitmq
spec:
 capacity:
   storage: 10Gi
 accessModes:
 - ReadWriteOnce
 persistentVolumeReclaimPolicy: Retain
 storageClassName: local-rabbitmq-storage
 local:
   path: /home/actmember/workspace/rabbitmq/data
 nodeAffinity:
   required:
     nodeSelectorTerms:
     - matchExpressions:
       - key: kubernetes.io/hostname
         operator: In
         values:
         - act-kube-01

```

# 3. Service 만들기

- rabbitmq-svc.yml

RabbitMQ와 관리를 위한 management console service를 생성합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  # Expose the management HTTP port on each node
  name: rabbitmq-management
  labels:
    app: rabbitmq
spec:
  ports:
  - port: 15672
    name: http
  selector:
    app: rabbitmq
  type: NodePort # Or LoadBalancer in production w/ proper security
---
apiVersion: v1
kind: Service
metadata:
  # The required headless service for StatefulSets
  name: rabbitmq
  labels:
    app: rabbitmq
spec:
  ports:
  - port: 5672
    name: amqp
  - port: 4369
    name: epmd
  - port: 25672
    name: rabbitmq-dist
  clusterIP: None
  selector:
    app: rabbitmq
```

# 4. StatefulSet 만들기

- rabbitmq-stateful.yml

본 파일은 StatefulSet을 이용하여 RabbitMQ Pod을 생성하고 있습니다.

RabbitMQ 도커 이미지를 실행할 때 사용할 하나 이상의 환경변수를 containers.env 속성으로 설정할 수 있습니다.

여러 환경변수값 중 RABBITMQ_ERLANG_COOKIE 속성값은 반드시 설정해주어야 합니다.

따라서 아래와 같이 StatefulSet 혹은 Pod을 생성할 때 **containers.env.name: RABBITMQ_ERLANG_COOKIE**
을 반드시 선언해야 합니다.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: "rabbitmq"
  replicas: 1
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      containers:
      - name: rabbitmq
        image: rabbitmq:3.7.7-management-alpine
        lifecycle:
          postStart:
            exec:
              command:
              - /bin/sh
              - -c
              - >
                if [ -z "$(grep rabbitmq /etc/resolv.conf)" ]; then
                  sed "s/^search \([^ ]\+\)/search rabbitmq.\1 \1/" /etc/resolv.conf > /etc/resolv.conf.new;
                  cat /etc/resolv.conf.new > /etc/resolv.conf;
                  rm /etc/resolv.conf.new;
                fi;
                until rabbitmqctl node_health_check; do sleep 1; done;
                if [[ "$HOSTNAME" != "rabbitmq-0" && -z "$(rabbitmqctl cluster_status | grep rabbitmq-0)" ]]; then
                  rabbitmqctl stop_app;
                  rabbitmqctl join_cluster rabbit@rabbitmq-0;
                  rabbitmqctl start_app;
                fi;
                rabbitmqctl set_policy ha-all "." '{"ha-mode":"exactly","ha-params":3,"ha-sync-mode":"automatic"}'
        env:
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbitmq-config
              key: erlang-cookie
        ports:
        - containerPort: 5672
          name: amqp
        volumeMounts:
        - name: rabbitmq
          mountPath: /var/lib/rabbitmq
  volumeClaimTemplates:
  - metadata:
      name: rabbitmq
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: local-rabbitmq-storage
      resources:
        requests:
          storage: 1Gi # make this bigger in production
```

# 5. 실행

위의 yml을 순차적으로 실행합니다.

~~~bash
$ kubectl create -f rabbitmq-pv.yml
$ kubectl create -f rabbitmq-svc.yml
$ kubectl create -f rabbitmq-stateful.yml
~~~

만약 성공적으로 실행이 되었다면 대쉬보드 또는 아래의 명령어를 통해서 팟이 실행된 것을 확인 할 수 있습니다.

~~~bash
$ kubectl get svc
~~~

다음 명령을 통해 rabbitmq-management 서비스의 port를 확인 하여 management console 에 접속하여 확인할 수 있습니다.
~~~bash
$ kubectl get svc
~~~
