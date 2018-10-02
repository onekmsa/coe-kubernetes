# Kubernetes Deploy
### Docker Registry Secret 설정
프라이빗 도커 레지스트리에서 이미지를 받아오기 위해서는 Secret설정이 필요합니다.  
아래와 같이 Secret을 생성한 후 ServiceAccount에 적용합니다.
```
$ kubectl create secret docker-registry coe-registry-key --docker-server=https://docker.sds-act.com --docker-username=dockeruser --docker-password=yourPassword

$ kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "coe-registry-key"}]}'
```
### Deployment Yaml 파일 작성
```yaml
# contents-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contents
  labels:
    app: contents
spec:
  replicas: 1
  selector:
    matchLabels:
      app: contents
  template:
    metadata:
      labels:
        app: contents
    spec:
      containers:
      - name: contents
        image: docker.sds-act.com/contents:latest
        ports:
        - containerPort: 5000
```
### Service Yaml 파일 작성
```yaml
# contents-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: contents
spec:
  selector:
    app: contents
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
  type: NodePort  # K8S 외부에서 접속 확인을 위해 NodePort로 설정하였으나
                  # 내부 서비스 호출만 일어날 경우 ClusterIP로 변경합니다.(미입력시 default: ClusterIP)
```   

### 생성
```sh
$ kubectl create -f contents-deployment.yaml
$ kubectl create -f contents-svc.yaml
```

### 확인 
![](../../../image/k8s-contents-deploy.png)
