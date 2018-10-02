# Mongodb 인스턴스 올리기

mongodb 단일 인스턴스를 StatefulSet을 이용하여 올리도록 하겠습니다.

쿠버네티스 환경(minikube, kubeadm 등)이 구축 되었다고 가정하고 진행하도록 하겠습니다.

# 1. StatefulSet 만들기

> 하단 yml 파일의 컨테이너 중 mongo-k8s-sidecar의 경우 복제본 세트 설정을 도와주는 사이드카 컨테이너입니다. 따라서 제거하고 실행 하여도 문제가 없습니다.


~~~yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  updateStrategy:
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        role: "mongo"
        environment: development
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
        - name: proposal-mongo-sidecar
          image: austbot/mongo-k8s-sidecar
          env:
            - name: KUBERNETES_MONGO_SERVICE_NAME
              value: "mongo"
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=development"
~~~


위와 같은 내용으로 파일(mongo-stateful.yml)을 만들고 아래의 명령으로 실행하여 StatefulSet 리소스를 생성합니다.

~~~bash
$ kubectl create -f mongo-stateful.yml
~~~

위 설정의 문제는 볼륨에 대한 마운팅이 없기 때문에 StatefulSet이 완전히 종료 되면 기존에 mongo에 저장 되었던 데이터가 사라지는 문제가 있습니다.

# 2. PersistentVolume 만들기

데이터를 유지하기 위해서 PersistentVolume 리소스를 아래와 같이 생성합니다.

해당 예제는 볼륨을 local에 지정하였습니다.

만약 aws, glusterfs 등을 이용하여 다이나믹 프로비져닝을 할 경우에는 쿠버네티스 홈페이지를 참조하십시오. ![바로가기](https://kubernetes.io/docs/concepts/storage/volumes/)

~~~yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /home/docker/test
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
~~~


> 위 설정 중 마지막의 minikube는 해당 볼륨이 생성될 노드의 이름입니다.
노드의 이름은 아래의 명령어로 확인 가능합니다.
~~~bash
$ kubectl get nodes
~~~
그리고 path는 마운팅 될 저장소의 경로입니다.

# 3. StatefulSet 수정

위의 StatefulSet 설정을 아래와 같이 수정합니다.

~~~yml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  updateStrategy:
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        role: "mongo"
        environment: development
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: local-claim
              mountPath: /data/db
        - name: proposal-mongo-sidecar
          image: austbot/mongo-k8s-sidecar
          env:
            - name: KUBERNETES_MONGO_SERVICE_NAME
              value: "mongo"
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=development"
  volumeClaimTemplates:
  - metadata:
     name: local-claim
    spec:
     accessModes:
      - ReadWriteOnce
     storageClassName: local-storage
     resources:
      requests:
        storage: 500Gi
~~~

# 4. 실행

위의 yml파일을 순차적으로 실행합니다.

~~~bash
$ kubectl create -f [파일명].yml
~~~

만약 성공적으로 실행이 되었다면 대쉬보드 또는 아래의 명령어를 통해서 팟이 실행된 것을 확인 할 수 있습니다.

~~~bash
$ kubectl get pods
~~~

다음 몽고디비에 접속하여 데이터를 넣도록 합니다.
~~~bash
$ kubectl exec -it mongo-0 mongo
rs0:PRIMARY> use test
switched to db test
rs0:PRIMARY> db.book.insert({"name": "act"})
WriteResult({ "nInserted" : 1 })
~~~

볼륨이 마운팅이 정상적으로 되었다면 해당 팟이 죽고 다시 재 실행이 되어도 해당 데이터가 유지가 되어야 합니다.

kubectl delete 명령어를 통해서 강제로 StatefulSet이나 팟을 죽인 이후에 다시 몽고디비에 접속합니다.

아래와 같은 명령어를 통해서 위에서 입력한 데이터가 살아 있는지 확인합니다.
~~~bash
$ kubectl exec -it mongo-0 mongo
rs0:PRIMARY> use test
switched to db test
rs0:PRIMARY> db.book.find()
{ "_id" : ObjectId("5b97887aa690c7d67983cfe8"), "name" : "act" }
~~~

# 5. 기타

## 5.1 PVC:PV = 1:1
동적 프로비져닝이 아닌 볼륨이 아닌 상태에서

StatefulSet의 replica를 증가 시킬 경우

해당 replica의 수 만큼 PersistentVolume을 만들어줘야 합니다.

이유는 Volume claim template 을 이용 할 경우 복제된 팟의 수만큼 PVC가 자동 생성이 되는데 PVC와 PV는 1:1 매핑이기 때문입니다.


# 5.2 Headless-Service 만들기

쿠버네티스의 서비스에서 clusterIP를 None으로 설정하면 헤드리스 서비스가 됩니다.

로드 밸런싱과 단일 서비스 IP가 필요치 않은 경우가 존재합니다.

이런 경우 헤드리스 서비스를 만들어야 합니다.

 StatefulSet 안의 확장 된 팟에서 서로의 데이터를 특정 이유(데이터 복제 등)로 인해서 바라봐야 할 때 기존의 서비스명을 사용하게 되면 서비스 안의 랜덤한 팟을 리턴하기 때문에 문제가 생길 수 있습니다.

헤드리스 서비스를 이용하면 아래와 같이 서비스안의 특정 팟에 직접 접근이 가능합니다.

~~~
<StatefulSet이름>-<순번>.<서비스명>
~~~

따라서 같은 StatefulSet내의 팟 사이에서 직접적인 통신이 필요할 경우 아래와 같이 헤드리스 서비스를 생성합니다.

~~~yml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
~~~
