# deployments
쿠버네티스는 객체(object) 저장소이고 객체와 상호 작용하는 코드입니다.

저장되는 각 객체에는 메타데이터, 스펙 및 현재 상태 이렇게 3가지가 존재합니다.

사용자는 메타데이터와 객체의 원하는 상태를 설명하는 사양을 제공합니다.

# 1. deployments란
쿠버네티스 deployment는 상태 저장 서비스를 관리하는 StatefulSets과 달리 클러스터에서 실행 중인 stateless 서비스를 관리합니다.

목적은 일련의 동일한 팟을 계속 실행하고 설정된 방식으로 업그레이드를 하는 것입니다.(기본적으로는 롤링 업데이트를 수행합니다.)

![](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/07751442-deployment.png)

# 2. deployment 생성
minikube 혹은 kubeadm 등으로 구성 된 클러스터 환경에서 실행을 하도록 합니다.

기본적인 실행 명령은 다음과 같습니다.(물론 kubectl create, apply 명령으로 json이나, yaml을 실행 할 수 있습니다.)
~~~bash
$ kubectl run act-httpd --image=httpd:latest --port 80
~~~

yaml 설정파일로 Deployment를 정의 후 적용할수 있습니다.  
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test-dep-kube-leo
  labels:
    app: test-kube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-kube
  template:
    metadata:
      labels:
        app: test-kube
        version: v17
    spec:
      containers:
      - name: test-kube
        image: docker.sds-act.com/test-dep-kube:17
        ports:
        - containerPort: 9000
      imagePullSecrets:
      - name: act-docker-registry-key
```
~~~bash
$ kubectl apply -f deployment.yaml
~~~

# 3. metadata 확인
실행중인 deployments를 확인하는 방법은 아래와 같습니다.
~~~bash
$ kubectl get deployments
~~~

만약 특정 이름의 deployments를 확인하려면 아래와 같이 입력합니다.
~~~bash
$ kubectl get deployments/act-httpd
~~~

만약 해당 정보를 yaml 또는 json형식으로 자세히 보고 싶으면 아래와 같이 입력합니다.
~~~bash
$ kubectl get deployments/act-httpd -o yaml
$ kubectl get deployments/act-httpd -o json
~~~

위 명령어를 입력하면 메타데이터, 스펙, 상태등의 정보를 볼 수 있는데 메타데이터부터 확인하겠습니다.

~~~bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: 2018-07-26T03:32:01Z
  generation: 1
  labels:
    run: act-httpd
  name: act-httpd
  namespace: default
  resourceVersion: "105416"
  selfLink: /apis/extensions/v1beta1/namespaces/default/deployments/act-httpd
  uid: 758c8468-9084-11e8-9747-24f5aac9a1ad
...
~~~

메타 데이터에는 deployment의 이름(유니크 해야 함), 쿠버네티스 내부에서 사용되는 uid 및 어노테이션 객체가 포함됩니다.

여기에는 하나의 어노테이션과 현재 deployment의 revision 정보를 확인 할 수 있습니다.

쿠버네티스의 각 객체는 key-value 쌍인 label 집합을 가질 수 있습니다.

## 4. spec 확인
~~~bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  ...
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      run: act-httpd
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: act-httpd
    spec:
      containers:
      - image: httpd:latest
        imagePullPolicy: Always
        name: act-httpd
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
~~~

spec에는 반드시 설정 되어야 할 두가지 key가 존재합니다.

* replicas - 해당 deployment에 얼마나 많은 팟이 있어야 하는지를 나타냅니다. 이 경우에는 현재 하나의 팟이 존재합니다.

* template - 각각의 팟에 대한 설명을 나타냅니다. 즉 팟에 존재하는 컨테이너 목록을 나타내고 있습니다.

다음 두가지 설정을 이용하여 deployment 동작을 정의 할 수 있습니다.

* selector - 해당 deployment의 일부로 간주되는 팟을 정합니다.

* strategy - 해당 deployment가 어떻게 롤아웃 하는 방법을 나타냅니다.


## 5. status 확인
~~~bash
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  ...
spec:
  ...
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: 2018-07-26T03:32:07Z
    lastUpdateTime: 2018-07-26T03:32:07Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2018-07-26T03:32:01Z
    lastUpdateTime: 2018-07-26T03:32:07Z
    message: ReplicaSet "act-httpd-d6f7b7f54" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1
~~~

마지막 부분의 status는 쿠버네티스가 관찰한 상태이며 쿠버네티스 스스로만 업데이트 할 수 있습니다.

여기서 replicas는 ReplicaSets 수가 아닌 복제 된 팟의 수를 나타냅니다.

replicas는 단순히 spec에서 복제 된 값입니다. 다만 비동기식으로 발생하기 때문에 spec.replicas와 status.replicas는 같지 않을 수 있습니다.

## 6. ReplicaSet object

ReplicaSet은 차세대 복제 컨트롤러(Replication-controller)입니다. 차이점은 selector지원 여부 입니다.

일반적으로 복제 컨트롤러를 지원하는 대부분의 kubectl 명령은 ReplicaSet도 지원합니다. 다만 다른 점은 롤링 업데이트 명령입니다.

만약 롤링업데이트 기능을 원한다면 deployment를 사용하길 바랍니다.

롤링 업데이트 기능은 필수적이며 deployment는 선언적이기 때문에 rollout 명령을 사용하는 것이 좋습니다.

ReplicaSet은 독립적으로 사용될 수 있지만 팟 생성, 삭제 및 업데이트를 조율하는 메커니즘으로는 deployment가 주로 사용됩니다.

deployment가 생성 되면 ReplicaSet도 만들어지기 때문에 걱정할 필요는 없습니다.

deployment가 replicaset을 관리하고 기타 유용한 기능과 함께 팟에 선언적 업데이트를 하기 때문에 deployment를 사용하는 것이 좋습니다. 즉 실제로는 ReplicaSets 객체를 조작 할 필요가 없다는 것을 의미합니다. deployment의 spec 섹션에서 정의하는 것이 좋습니다.


deployment의 metadata 영역에서 label을 이용하여 ReplicaSet을 확인 할 수 있습니다.

위에서 보인 label의 경우 run: act-httpd 이기 때문에 아래와 같이 입력해줍니다.

~~~bash
$ kubectl get rs -l run=act-httpd
NAME                  DESIRED   CURRENT   READY     AGE
act-httpd-d6f7b7f54   1         1         1         52m
~~~

이제 해당 RS의 상세 정보를 확인합니다.

~~~bash
$ kubectl get rs act-httpd-d6f7b7f54 -o yaml
~~~

~~~bash
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "1"
    deployment.kubernetes.io/max-replicas: "2"
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: 2018-07-26T03:32:01Z
  generation: 1
  labels:
    pod-template-hash: "829363910"
    run: act-httpd
  name: act-httpd-d6f7b7f54
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: act-httpd
    uid: 758c8468-9084-11e8-9747-24f5aac9a1ad
  resourceVersion: "105415"
  selfLink: /apis/extensions/v1beta1/namespaces/default/replicasets/act-httpd-d6f7b7f54
  uid: 758d8db0-9084-11e8-9747-24f5aac9a1ad
spec:
  replicas: 1
  selector:
    matchLabels:
      pod-template-hash: "829363910"
      run: act-httpd
  template:
    metadata:
      creationTimestamp: null
      labels:
        pod-template-hash: "829363910"
        run: act-httpd
    spec:
      containers:
      - image: httpd:latest
        imagePullPolicy: Always
        name: act-httpd
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 1
  fullyLabeledReplicas: 1
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
~~~

deployment 정보와 비슷한 화면을 확인 할 수 있습니다.

ReplicaSets은 실제로 run=act-httpd 라는 label을 가지고 있습니다. 그리고 추가된 label이 있는데 이렇게 2가지 label을 가지고 팟의 정보를 조회 할 수 있습니다.

~~~bash
$ kubectl get pods -l "run=my-nginx,pod-template-hash=829363910"
~~~

~~~bash
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    creationTimestamp: 2018-07-26T03:32:01Z
    generateName: act-httpd-d6f7b7f54-
    labels:
      pod-template-hash: "829363910"
      run: act-httpd
    name: act-httpd-d6f7b7f54-vj5lh
    namespace: default
    ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
      kind: ReplicaSet
      name: act-httpd-d6f7b7f54
      uid: 758d8db0-9084-11e8-9747-24f5aac9a1ad
    resourceVersion: "105414"
    selfLink: /api/v1/namespaces/default/pods/act-httpd-d6f7b7f54-vj5lh
    uid: 758f7092-9084-11e8-9747-24f5aac9a1ad
  spec:
    containers:
    - image: httpd:latest
      imagePullPolicy: Always
      name: act-httpd
      ports:
      - containerPort: 80
        protocol: TCP
      resources: {}
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      volumeMounts:
      - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
        name: default-token-58tqc
        readOnly: true
    dnsPolicy: ClusterFirst
    nodeName: kwang-870z5g-880z5f
    priority: 0
    restartPolicy: Always
    schedulerName: default-scheduler
    securityContext: {}
    serviceAccount: default
    serviceAccountName: default
    terminationGracePeriodSeconds: 30
    tolerations:
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
    volumes:
    - name: default-token-58tqc
      secret:
        defaultMode: 420
        secretName: default-token-58tqc
  status:
    conditions:
    - lastProbeTime: null
      lastTransitionTime: 2018-07-26T03:32:01Z
      status: "True"
      type: Initialized
    - lastProbeTime: null
      lastTransitionTime: 2018-07-26T03:32:07Z
      status: "True"
      type: Ready
    - lastProbeTime: null
      lastTransitionTime: null
      status: "True"
      type: ContainersReady
    - lastProbeTime: null
      lastTransitionTime: 2018-07-26T03:32:01Z
      status: "True"
      type: PodScheduled
    containerStatuses:
    - containerID: docker://bfb7fc5befa8a808b34948648cb5812dfd2c9cdd96deabcee63cb303a2a36852
      image: httpd:latest
      imageID: docker-pullable://httpd@sha256:2edbf09d0dbdf2a3e21e4cb52f3385ad916c01dc2528868bc3499111cc54e937
      lastState: {}
      name: act-httpd
      ready: true
      restartCount: 0
      state:
        running:
          startedAt: 2018-07-26T03:32:07Z
    hostIP: 192.168.20.230
    phase: Running
    podIP: 10.40.0.3
    qosClass: BestEffort
    startTime: 2018-07-26T03:32:01Z
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
~~~

위에서 보다시피 label의 값은 ReplicaSet template에서 지정된 것과 동일합니다. 그리고 컨테이너의 사양은 deployment의 템플릿에서 ReplicaSet을 통해 Pod으로 전해진 것을 확인 할 수 있습니다.

## 출처
https://thenewstack.io/kubernetes-deployments-work/

https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
