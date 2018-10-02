# Redis 인스턴스 올리기

redis 단일 인스턴스를 StatefulSet을 이용하여 올리도록 하겠습니다.

쿠버네티스 환경(minikube, kubeadm 등)이 구축 되었다고 가정하고 진행하도록 하겠습니다.

# 1. PersistentVolume 만들기

데이터를 지속적으로 유지하기 위해서 PersistentVolume 리소스를 아래와 같이 생성합니다.

해당 예제는 볼륨을 node의 local에 지정하였습니다.

만약 aws, glusterfs 등을 이용하여 다이나믹 프로비져닝을 할 경우에는 쿠버네티스 홈페이지를 참조하십시오. [바로가기](https://kubernetes.io/docs/concepts/storage/volumes/)

~~~yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-redis
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/redis   
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - act-kube-03
~~~


> 위 설정 중 마지막의 act-kube-03는 해당 볼륨이 생성될 노드의 이름입니다.  
노드의 이름은 아래의 명령어로 확인 가능합니다.
~~~bash
$ kubectl get nodes
~~~
그리고 path는 마운팅 될 저장소의 경로입니다.  
PV 생성 시에는 경로가 없어도 에러가 나지 않지만, container 실행시 volume 매핑이 안되어 에러가 발생하게 됩니다.  
storageClassName는 PV들을 구분하는 기준으로 claim에서 사용 됩니다.  

위와 같은 내용으로 파일을 만들고 아래의 명령으로 실행하여 PersistentVolume 리소스를 생성합니다.  

모든 yml 파일은 아래 명령어로 생성하면 됩니다.  

~~~bash
$ kubectl apply -f [파일명].yml
~~~


# 2. ConfigMap으로 redis.conf 만들기

ConfigMap을 생성하여 redis에서 사용할 redis.conf 를 적용할 수 있습니다.  

redis.conf 를 미리 정의하여 준비해 둡니다.  

아래 명령어로 redis-config라는 이름의 configmap을 생성합니다.  

~~~bash
$ kubectl create cm redis-config --from-file=redis.conf
~~~

# 3. StatefulSet 만들기

이미 생성한 PersistentVolume와 ConfigMap 을 이용하여 Redis StatefulSet을 만들도록 하겠습니다.  

[bitnami/redis](https://hub.docker.com/r/bitnami/redis/) 의 이미지를 사용하여 redis.conf 전체를 configmap으로 대체할 수 있고, 비밀번호를 container의 env 값으로 설정 할 수 있습니다.

아래 yaml은 ConfigMap을 사용하여 redis.conf를 적용하고 있습니다.  

~~~yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-master
spec:
  serviceName: redis-master
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: bitnami/redis:latest
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
        volumeMounts:
        - mountPath: /bitnami/redis/data
          name: local-claim
        - mountPath: /opt/bitnami/redis/etc
          name: redisconfig
      volumes:
        - name: redisconfig
          configMap:
            name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: local-claim
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: local-storage
      resources:
        requests:
          storage: 1Gi
~~~

# 4. 서비스 확인

StatefulSet과 모든 오브젝트가 정상 생성되었는지 먼지 확인 합니다.

```bash
$ kubectl get pv,pvc,pod,sts -o wide
```

일부 오브젝트의 상태가 정상이 아니라면 아래 명령어로 event를 조회 할 수 있습니다.  
(pod redis-master-0의 이벤트 조회)
```bash
$ kubectl describe pod/redis-master-0
```

그리고 redis가 정상적으로 실행되는지 확인하기 위해 container 내부로 접속합니다.  

아래 명령어를 실행하여 OK 결과가 표시되면 정상입니다.  

~~~bash
$ kubectl exec -it your-pod-name -- /bin/bash

접속 후
redis-cli
>> AUTH your-yourPassword
OK
~~~



# 5. password 설정을 위해 env를 사용하는 경우

ConfigMap이 아닌 secret을 만들어 container의 env로 전달하는 방법에 대한 설명 입니다.  

아래 명령어로 secret을 생성합니다.  

~~~bash
kubectl create secret generic redis-password --from-literal=REDIS-PASSWORD=your-password
~~~

이 secret을 사용하여 StatefulSet을 생성 합니다.

~~~yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-master
spec:
  serviceName: redis-master
  selector:
    matchLabels:
      app: redis
      role: master
      tier: backend
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
        role: master
        tier: backend
    spec:
      containers:
      - name: master
        image: bitnami/redis:latest
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 6379
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-password
              key: REDIS-PASSWORD
        volumeMounts:
        - mountPath: /bitnami/redis/data
          name: local-claim
  volumeClaimTemplates:
  - metadata:
      name: local-claim
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: local-storage
      resources:
        requests:
          storage: 1Gi

~~~

# 6. TODO:
password를 secret 로 전달하고 configmap과 같이 사용하면 configmap의 설정 내용으로 적용되어 password 전달이 무시되고 있음  
이에 대한 추가적인 테스트 필요  
