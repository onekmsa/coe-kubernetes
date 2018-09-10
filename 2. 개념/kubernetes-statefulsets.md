# StatefulSet을 통한 Replication

ReplicaSets은 Pod 템플릿을 통해 여러개의 Replica를 구성합니다. ReplicaSet으로 구성된 Pod들은 서로 다른 이름과 아이피 주소를 갖습니다.   

만약 Pod 템플릿에 PersistentVolumeClaim(PVC)을 통해 볼륨을 맵핑할 경우 모든 Replica들은 동일한 PVC를 통해 볼륨을 요청하고  
동일한 PersistentVolume(PV)에 바운딩 됩니다. 즉, ReplicaSets은 모든 Pod들이 동일한 스토리지 볼륨을 사용합니다.   

Pod이 죽어서 ReplicaSet이 새로운 Pod으로 교체할 때, 교체된 Pod은 새로운 호스트네임과 아이피를 가지는 완전히 새로운 Pod입니다.  
하지만 특정 어플리케이션의 경우는 새로운 Pod으로 교체되더라도 동일한 호스트네임과 아이피가 필요합니다.  




# Kubernetes Internal  
Control Plane(Master node)  
- etcd  
- Scheduler
- Controller Manager
- API Server
Work Node
- kube-proxy
- kubelet
- controller runtime

모든 컴포넌트들은 API server를 통해서만 통신할 수 있습니다. 즉, 다른 컴포넌트에 직접 요청할 수 없습니다.
API server는 etcd와 통신할 수 있는 유일한 컴포넌트이고 다른 컴포넌트들은 etcd에 직접 접근할 수 없습니다.   

대부분의 컴포넌트들은 컴포넌트에서 API Server에 요청을 보냄으로써 커넥션이 이루어지지만,   
클라이언트가 kubectl 명령어를 실행하면 API server는 Kubelet에 접근할 수 있습니다.   

work node에 있는 컴포넌트들은 동일한 노드에서 실행되어야 하지만 control pane의 컴포넌트들은 다중 서버로 분산될 수 있습니다.  
고가용성을 위해 etct와 API server는 다중 인스턴스를 실행시켜 병렬로 처리할 수 있지만,  
Scheduler 와 Controller Manager는 단일 인스턴스로만 활성화 됩니다.  

Control Plane 컴포넌트들과 kube-proxy는 시스템상에 직접 배포되거나 Pod으로 실행될 수 있습니다.  
Kubelet은 모든 다른 컴포넌트들을 Pod으로 실행시킬 수 있는 컴포넌트입니다. 따라서 Control Plane 컴포너트들을 Pod으로 실행시키려면 마스터 노드에도 Kubelet이 배포되어야 합니다.

### how kubernetes uses etcd
Pods, ReplicationControllers, Services 등과 같은 모든 객체들은 어딘가에 저장이 되어 있어야 시스템이 재구동 되어도 다시 사용할 수 있습니다. 이를 위해서 Kubernetes는 key-value 스토어인 etcd를 사용합니다.  
etcd는 고가용성을 위해 다중 인스턴스로 실행시킬 수 있습니다.  
