# 5% - scheduling

- Use label selectors to schedule Pods.
- Understand the role of DaemonSets.
- Understand how resource limits can affect Pod scheduling.
- Understand how to run multiple schedulers and how to configure Pods to use them.
- Manually schedule a pod without a scheduler.
- Display scheduler events.
- Know how to configure the Kubernetes scheduler.

## Use label selectors to schedule Pods.
- nodeSelector
Pod의 spec.nodeSelector에 node 오브젝트의 label 정보를 명시하여 어떤 node에 스케줄링할 지 선택할 수 있습니다.

```sh
$ kubectl label nodes ssd-node disktype=ssd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

- Node affinity (beta)

spec.affinity에 nodeAffinity를 설정할 수 있습니다.
| Field                                            | 명시한 Label Node에 반드시 할당 | Node Label 변경 시 Pod evit 여부 |
|--------------------------------------------------|---------------------------------|----------------------------------|
| requiredDuringSchedulingRequiredDuringExecution  | O                               | O                                |
| requiredDuringSchedulingIgnoredDuringExecution   | O                               | X                                |
| preferredDuringSchedulingRequiredDuringExecution | X                               | O                                |
| preferredDuringSchedulingIgnoredDuringExecution  | X                               | X                                |
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2     #either e2e-az1 or e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1  # 1-100 range. 일치하는 node에 합산하여 가장 숫자가 높은 node 선택  
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

## Understand the role of DaemonSets.
DaemonSet은 클러스터링되어 있는 각 Node에서 Pod을 실행시킵니다. 새로운 Node가 클러스터에 추가되면 DaemonSet에 정의된 Pod이 실행됩니다.  
ex) fluentd나 logstash 같은 모든 노드에서 실행되어야 하는 로그 콜렉터

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
```

![](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)

# Reference

## Label
### Equality-based requirement (=, ==, !=)
- URL query string
```
?labelSelector=environment%3Dproduction,tier%3Dfrontend
```
- kubectl
```sh
$ kubectl get pods -l environment=production,tier=frontend
```

### Set-based requirement (in, notin, exists)
- URL query string
```
?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29
```
- kubectl
```sh
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
$ kubectl get pods -l 'environment in (production, qa)'
$ kubectl get pods -l 'environment,environment notin (frontend)'
```

* Service, ReplicationController는 Equality-based requirement selector만 가능
```yaml
selector:
  componenet: redis
```

* Job, Deployment, Replica Set, Daemon Set은 Set-based requirement selector도 가능  
```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```
