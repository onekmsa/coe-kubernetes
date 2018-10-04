# k8s Logging

로깅이 필요한 이유
https://kubernetes.io/docs/concepts/cluster-administration/logging/



단일 팟 로깅
- 표준출력
- 파일 복사 등

중앙 집중 로깅
- EFK
- Daemonset

로그는 쿠버네티스 상에서 일어나는 일을 기록하고 해당 로그를 통해서 발생한 문제들을 해결 할 수 있습니다.

컨테이너 환경에서 일반적인 로깅 방법은 표준 출력과 표준 에러 스트림을 이용하여 기록하는 것 입니다.

팟에 직접 접근하여 로그를 확인하는 방법부터 클러스터 레벨에서 로깅을 처리하는 방법을 확인하도록 하겠습니다.

## 1. 기본적인 로깅

### 1.1 kubectl log

kubectl log 명령어를 이용할 경우 컨테이너에서 발생한 표준 출력에 대한 정보를 확인 할 수 있습니다.

컨테이너에서 발생한 표준 출력의 경우 각 노드의 특정 경로에 해당 로그 정보가 쌓이게 됩니다. (운영체제마다 경로는 다를 수 있고 팟이 죽을 경우 해당 로그 파일은 삭제 됨)

아래와 같은 팟을 실행하고

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
~~~

아래의 명령어로 로그를 확인합니다.

~~~bash
$ kubectl logs -f counter
0: Tue Oct  2 08:24:55 UTC 2018
1: Tue Oct  2 08:24:56 UTC 2018
2: Tue Oct  2 08:24:57 UTC 2018
3: Tue Oct  2 08:24:58 UTC 2018
...
~~~

해당 팟이 실행중인 노드(ubuntu 18.04)의 /var/log/containers 경로에서 해당 팟의 로그 파일을 확인 할 수 있습니다.
~~~bash
$ cd /var/log/containers
$ ls -al
counter_default_count-a273429e38484bc9ef37abe9f3ea1881581524bd2a4278c4e66736db7a6e2c9c.log
~~~

만약 팟을 삭제 할 경우 위의 파일이 삭제 됨을 알 수 있습니다.

### 1.2 kubectl exec

만약 실행중인 어플리케이션이 표준 출력 없이 파일에 로그를 기록하는 형태라면 아래의 명령어로 해당 팟의 컨테이너로 이동하여 로그를 확인합니다.

~~~bash
$ kubectl exec [POD_NAME] cat [LOG_FILE]
~~~

만약 팟 안에 여러 컨테이너가 있을 경우 아래와 같은 명령어로 컨테이너를 지정합니다.

~~~bash
$ kubectl exec [POD_NAME] -c [CONTAINER_NAME] cat [LOG_FILE]
~~~

### 1.3 kubectl cp

팟 로그의 경우 해당 로그 파일에 대한 볼륨을 별도로 지정하지 않을 경우 팟이 삭제 될 경우 로그 파일이 사라지게 됩니다.

혹은 본인의 작업 pc에서 로그 분석이 편할 경우 해당 로그 파일을 복사하도록 합니다.

~~~bash
$ kubectl cp [POD_NAME]:[SOURCE_PATH] [TARGET_PATH]
~~~

## 2. 노드 레벨에서의 로깅 고려 사항

위에서 설명 하였듯이 응용프로그램에서 발생하는 로그는 표준출력 또는 표준에러를 통해서 처리되고 이 과정은 컨테이너 엔진에 의해서 어디론가 재 지정되고 있습니다.

팟 안의 컨테이너가 종료되면 kubelet은 로그와 함께 컨테이너를 보관하고 팟이 제거되면 로그와 함께 팟 안의 컨테이너는 사라지게 됩니다.

즉, 팟이 사라지지 않는 이상 컨테이너에서 발생된 로그는 지속적으로 어딘가에 쌓이고 있습니다.

로그가 중요하지만 로그가 노드의 저장소를 모두 소모하지 않도록 제어를 해줄 필요가 있습니다.

쿠버네티스 클러스터에는 매 시간 실행하도록 구성 된 logrotate 도구가 존재합니다.

해당 도구를 이용하여 로그의 보관 주기, 파일의 갯수 등을 설정 할 수 있습니다.

만약 kubeup.sh 스크립트가 아닌 다른 방법으로 클러스터링을 하였다면 컨테이너 런타임의 옵션을 설정하여 해당 기능을 구현할 수 있습니다. 도커의 경우 도커 데몬에 아래의 옵션을 추가하면 가능합니다.

~~~
--log-driver=json-file --log-opt=max-size=10m --log-opt=max-file=5
~~~

## 3. 시스템 구성요소 로그

쿠버네티스 시스템 구성요소는 컨테이너로 실행 된 것(쿠버네티스 스케쥴러, 프록시)과 그렇지 않은 것(kubelet, 컨테이너 런타임)으로 구성 됩니다.

systemd가 실행되어 있는 머신이라면 kubelet과 컨테이너 런타임은 journald에 기록을 합니다.

systemd-journald는 로그 관리 데몬으로 부팅이 시작 되는 순간부터 로그를 수집합니다.

만약 systemd가 없으면 /var/log 디렉토리에 로그파일을 기록합니다.

컨테이너로 실행되지 않은 구성요소는 항상 /var/log 디렉토리에 기록을 합니다.

## 4. 클러스터 레벨에서의 로깅

쿠버네티스에서는 클러스터 수준의 로깅을 위한 별도의 솔루션을 제공하지 않지만 아래와 같이 몇 가지 일반적인 방법이 존재합니다.

* 모든 노드에서 실행되는 노드 레벨 로깅 에이전트 사용

* 팟 내부에 로깅을 위한 사이드카 컨테이너 포함

* 어플리케이션 내부로부터 로그를 백엔드로 직접 푸쉬

### 4.1 노드 레벨의 로깅 에이전트 사용

![](./img/node-agent-logging.png)

쿠버네티스 클러스터에 대해 가장 일반적이고 권장되는 방법으로 각 노드에 로깅 에이전트를 포함시켜 해당 에이전트가 모든 로그를 수집하여 노출시키거나 백엔드(잘 알려진 백엔드로는 엘라스틱서치)에 로그를 푸시하는 형태입니다.

이러한 로깅 에이전트는 노드에 있는 모든 로그 파일에 대한 액세스 권한이 존재합니다.

로깅 에이전트는 일반적으로 모든 노드에 존재하기 때문에 DaemonSet 리소스를 이용하여 만듭니다.

노드 당 하나의 에이전트만 만들고 기존의 응용프로그램들을 변경할 필요가 없다는 장점이 존재하지만 표준 출력과 표준 오류에 대해서만 작동한다는 단점이 존재합니다.

위의 그림을 EFK로 구성하게 되면 아래와 같은 그림으로 표현할 수 있습니다.

![](./img/efk.png)


* ElastciSearch(이하 엘라스틱서치) - 로그를 저장하고 쿼리를 통해서 정보를 가져올 수 있도록 하는 엔진입니다.

* fluentd - 쿠버네티스에서 엘라스틱서치로 로그 메시지를 보내는 에이전트 역할을 합니다.

* Kibana(이하 키바나) - 엘라스틱서치에 저장된 로그를 통해 사용자에게 그래픽 인터페이스를 제공하는 역할을 합니다.

아래 단계를 통해서 EFK를 설치하도록 합니다.

아래의 단계는 쿠버네티스에서 제공하는 기본적인 설치 단계이기 때문에 실제 운영시에는 Secret 및 Storage 설정을 반드시 하는 것이 좋습니다.


#### 4.1.1 엘라스틱서치

대부분의 저장 관련 솔루션이 그렇듯이 엘라스틱서치도 StatefulSet을 이용하여 쿠버네티스 상에서 실행합니다.

elasticsearch-logging이라는 ServiceAccount 리소스와 ClusterRole 리소스를 생성한 이후 ClusterRoleBinding 리소스를 통해 권한을 부여합니다.

elasticsearch-logging이라는 계정은 core API group를 사용하며 service, namespace, endpoint 리소스에 대한 get 권한을 가지게 됩니다.

~~~yaml
# RBAC authn and authz
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: elasticsearch-logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "services"
  - "namespaces"
  - "endpoints"
  verbs:
  - "get"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: kube-system
  name: elasticsearch-logging
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: elasticsearch-logging
  namespace: kube-system
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: elasticsearch-logging
  apiGroup: ""
~~~

아래와 같이 엘라스틱서치의 StatefulSet과 Service를 등록합니다.

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Elasticsearch"
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    k8s-app: elasticsearch-logging
---    
# Elasticsearch deployment itself
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    version: v6.2.5
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  serviceName: elasticsearch-logging
  replicas: 2
  selector:
    matchLabels:
      k8s-app: elasticsearch-logging
      version: v6.2.5
  template:
    metadata:
      labels:
        k8s-app: elasticsearch-logging
        version: v6.2.5
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccountName: elasticsearch-logging
      containers:
      - image: k8s.gcr.io/elasticsearch:v6.2.5
        name: elasticsearch-logging
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: elasticsearch-logging
          mountPath: /data
        env:
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
      - name: elasticsearch-logging
        emptyDir: {}
      # Elasticsearch requires vm.max_map_count to be at least 262144.
      # If your OS already sets up this number to a higher value, feel free
      # to remove this init container.
      initContainers:
      - image: alpine:3.6
        command: ["/sbin/sysctl", "-w", "vm.max_map_count=262144"]
        name: elasticsearch-logging-init
        securityContext:
          privileged: true
~~~


### 4.1.2 fluentd

fluentd는 로깅 에이전트 역할을 하게 됩니다. 따라서 모든 노드에서 실행 되어야 할 필요가 있습니다.

그렇기 때문에 해당 에이전트는 DaemonSet을 이용하여 실행합니다.

fluentd 에이전트를 사용하려면 먼저 모든 노드에 아래와 같은 레이블을 지정해줘야 합니다.

~~~
beta.kubernetes.io/fluentd-ds-ready=true
~~~

아래 명령어를 통해서 각 노드에 해당 레이블을 지정합니다.

~~~bash
$ kubectl label node [NODE_NAME] beta.kubernetes.io/fluentd-ds-ready=true
~~~


엘라스틱서치와 마찬가지로 ServiceAccount와 ClusterRole을 생성합니다.

~~~yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: kube-system
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  namespace: kube-system
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
~~~

fluentd 컨테이너가 사용할 ConfigMap을 생성합니다.

로그 수집에 관련된 각종 설정 정보가 저장되어 있기 때문에 운영 환경에 맞게 수정하도록 합니다. yml 파일의 내용이 크기 때문에 파일을 참조 하십시오.

ConfigMap을 생성하였으면 아래의 DaemonSet을 생성합니다.

하단의 volumeMounts 부분에서 노드에 저장되는 로그의 폴더가 매핑이 되어 있는 모습을 확인할 수 있습니다.

~~~yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-es-v2.2.0
  namespace: kube-system
  labels:
    k8s-app: fluentd-es
    version: v2.2.0
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-es
      version: v2.2.0
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
        kubernetes.io/cluster-service: "true"
        version: v2.2.0
      # This annotation ensures that fluentd does not get evicted if the node
      # supports critical pod annotation based priority scheme.
      # Note that this does not guarantee admission on the nodes (#40573).
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      priorityClassName: system-node-critical
      serviceAccountName: fluentd-es
      containers:
      - name: fluentd-es
        image: k8s.gcr.io/fluentd-elasticsearch:v2.2.0
        env:
        - name: FLUENTD_ARGS
          value: --no-supervisor -q
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /etc/fluent/config.d
      nodeSelector:
        beta.kubernetes.io/fluentd-ds-ready: "true"
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-es-config-v0.1.5
~~~


### 4.1.3 kibana

마지막으로 fluentd가 수집하고 엘라스틱서치에 저장된 로그 정보를 확인하기 위한 키바나를 실행합니다.

env 부분에서 엘라스틱서치를 바라보고 있는 것을 확인 할 수 있습니다.
~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: kibana-logging
  namespace: kube-system
  labels:
    k8s-app: kibana-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "Kibana"
spec:
  ports:
  - nodePort: 30400
    port: 5601
    protocol: TCP
    targetPort: ui
  selector:
    k8s-app: kibana-logging
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana-logging
  namespace: kube-system
  labels:
    k8s-app: kibana-logging
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: kibana-logging
  template:
    metadata:
      labels:
        k8s-app: kibana-logging
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'docker/default'
    spec:
      containers:
      - name: kibana-logging
        image: docker.elastic.co/kibana/kibana-oss:6.2.4
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch-logging:9200
          - name: SERVER_BASEPATH
            value: /api/v1/namespaces/kube-system/services/kibana-logging/proxy
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP
~~~

http://[클러스터 IP]:30400 에서 키바나 대쉬보드를 아래와 같이 확인 할 수 있습니다.

![](./img/kibana-dashboard.png)

로그가 정상적으로 보이는 것을 확인 하기 위해서 nginx를 실행합니다.
~~~bash
$ kubectl run nginx-test --image=nginx
~~~

그리고 해당 nginx 팟으로 접속을 시도 합니다.

~~~bash
$ curl $(kubectl get pods -l 'run=nginx-test'  -o jsonpath="{range .items[*]}{@.status.podIP}{end}")
~~~

키바나 화면에서 아래의 로그를 확인 할 수 있습니다.

![](./img/kibana-nginx.png)



### 4.2 사이드카 이용

사이드카를 이용한 로깅은 아래 방법 중 하나를 선택하여 사용 할 수 있습니다.

* 스트리밍 사이드카 컨테이너 이용

* 로깅 에이전트 사이드카 컨테이너 이용


#### 4.2.1 스트리밍 사이드카 컨테이너

![](./img/sidecar.png)

각 어플리케이션에 로그 스트림을 분산 할 필요가 존재한다면 별도의 스트리밍 사이드카 컨테이너를 이용하도록 합니다.

application 컨테이너에서는 표준 출력을 통해서 2개의 로그(ERR, ACCESS)를 볼륨에 파일을 작성하고 APP로그는 표준 출력으로 내보냅니다.

사이드카 컨테이너 (error-log, access-log)는

application 컨테이너에서 생성된 로그 파일을 자신의 스트리밍을 통해서 표준 출력으로 내보냅니다.

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
spec:
  containers:
  - name: application
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date) [APP]";
        echo "$i: $(date) [ERR]" >> /var/log/error.log;
        echo "$(date) INFO $i [ACCESS]" >> /var/log/access.log;
        i=$((i+1));
        sleep 5;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: error-log
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/error.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: access-log
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/access.log']
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  volumes:
  - name: varlog
    emptyDir: {}
~~~

해당 팟을 실행을 한 이후 아래 명령어를 실행하게 되면

메인 컨테이너인 application 컨테이너의 표준 출력 로그를 확인 할 수 있습니다.

~~~bash
$
1: Thu Oct  4 11:15:22 UTC 2018 [APP]
2: Thu Oct  4 11:15:27 UTC 2018 [APP]
3: Thu Oct  4 11:15:32 UTC 2018 [APP]
4: Thu Oct  4 11:15:37 UTC 2018 [APP]
...
~~~

마찬가지로 error-log 컨테이너와 access-log 컨테이너의 로그도 아래와 같이 확인이 가능합니다.

~~~bash
$ kubectl logs -f application -c error-log
0: Thu Oct  4 11:15:17 UTC 2018 [ERR]
1: Thu Oct  4 11:15:22 UTC 2018 [ERR]
2: Thu Oct  4 11:15:27 UTC 2018 [ERR]
3: Thu Oct  4 11:15:32 UTC 2018 [ERR]
4: Thu Oct  4 11:15:37 UTC 2018 [ERR]
...
~~~

~~~bash
$ kubectl logs -f application -c access-log
Thu Oct  4 11:15:17 UTC 2018 INFO 0 [ACCESS]
Thu Oct  4 11:15:22 UTC 2018 INFO 1 [ACCESS]
Thu Oct  4 11:15:27 UTC 2018 INFO 2 [ACCESS]
Thu Oct  4 11:15:32 UTC 2018 INFO 3 [ACCESS]
...
~~~

실제로 해당 팟이 실행중인 노드의 로그 경로로 가게 되면

아래와 같이 3개의 컨테이너 로그가 따로 만들어진 것을 확인 할 수 있습니다.

![](./img/sidecar-logfile.png)

클러스터에 설치된 로깅 에이전트는 추가 구성 없이 해당 로그 스트림을 가져 올 수 있습니다.

다만 로그를 파일에 기록한 다음 stdout으로 스트리밍하면 디스크 사용량이 증가 한다는 단점이 존재합니다.

#### 4.2.2 로깅 에이전트 사이드카 컨테이너

4.1 에서 설명한 노드 레벨의 로깅 에이전트 만으로 부족한 상황이면 팟 안에 별도의 로깅 에이전트를 사용하여 사이드카 컨테이너를 만들 수 있습니다.

다만 팟 마다 로깅 에이전트를 추가 할 경우 많은 리소스 소비가 있을 뿐더러 kubelet에 의해 제어 되지 않기 때문에 kubectl log 명령을 사용하여 해당 로그에 접근을 할 수 없습니다.





## 5. 참고 자료

https://kubernetes.io/docs/concepts/cluster-administration/logging/

https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch
