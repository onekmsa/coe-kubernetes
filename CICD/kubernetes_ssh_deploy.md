
## Jenkins

### Deploy with SSH

Jenkins 호스트 서버에서 docker agent를 실행시키고  
agent에서 Kubernetes 서버에 ssh로 직접 접속하여 배포하는 방법으로 아래의 pipeline으로 구성됩니다.   
 - Cloning Git
 - Build Project
 - Build Image
 - Push Image : private repository에 생성한 image를 push
 - Kubernetes Deploy : Kubernetes 서버에 ssh로 접속하여 kubectl 명령어로 image 교체 


1. Jenkins Plugin 설치
 - docker
 - ssh agent

2. Jenkins Credential 설정
Pipeline에서 사용할 인증 정보를 추가해 줍니다. 
 - git clone 시 사용할 인증 정보를 등록하여 Pipeline에 아래와 같이 credentialsId를 입력해 줍니다.  
   ```sh
   steps {
        git credentialsId: 'min0418', url: 'https://github.com/SDSACT/coe-eureka.git'
      }
   ```
 - private docker repository에 image를 푸시하기 위해 인증정보를 등록하여 Pipeline에 아래와 같이 credentialsId를 입력해 줍니다.  
   ```sh
   docker.withRegistry('https://docker.sds-act.com', 'dockeruser' ) {
        dockerImage.push()
   }
   ```
   
3. Kubernetes Secret 설정
Kubernetes 서버에서 Private docker registry로 부터 이미지를 pull 하기 위해 Secret 오브젝트를 생성하고  
default Serviceaccount에 적용해 줍니다.

```sh
kubectl create secret docker-registry actregistrykey --docker-server=https://docker.sds-act.com --docker-username=dockeruser --docker-password=yourpwd
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "actregistrykey"}]}'
```

4. Jenkins Pipeline 
```sh
def remote = [:]
remote.name = 'test'
remote.host = '192.168.10.77'
remote.user = 'actmember'
remote.password = 'yourpwd'
remote.allowAnyHosts = true

pipeline {
  agent {
    docker {
      image 'common/jenkins-slave-jdk-maven-git-docker:0.1'
      args '-v /maven-local-repository:/root/.m2/repository'
      registryCredentialsId 'dockeruser'
      registryUrl 'https://docker.sds-act.com'
    } 
  }
  
  environment {
    registry = "docker.sds-act.com/leo-test"
    dockerImage = ''
    buildnum = ''
  }
  
  stages {
    stage('Cloning Git') {
      steps {
        git credentialsId: 'min0418', url: 'https://github.com/SDSACT/coe-eureka.git'
      }
    }      
    stage('Build Project') {
      steps {
        script{
            sh "mvn clean install -Dprofile=kube -DskipTests=true"
        }
      }
    }          
    stage('Building image') {
      steps{
        script {
          buildnum = '${BUILD_NUMBER}'
          dockerImage = docker.build registry + ":" + buildnum
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry('https://docker.sds-act.com', 'dockeruser' ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Kubernetes Deploy') {
      steps{
        script {
          build_number = sh (
                script: "echo ${BUILD_NUMBER}",
                returnStdout: true
            )
          echo "${build_number}"
          sshCommand remote: remote, command:  'kubectl run leo-test --env="SPRING_PROFILES_ACTIVE=kube" --image=docker.sds-act.com/leo-test:' + build_number
        }
      }
    }
  }
}
```

### Deploy with JNLP

Jenkins의 Kubernetes Plugin을 통해 agent를 Kubernetes 서버에서 실행시킵니다.

1. Jenkins Plugin 설치
 - docker
 - kubernetes

2. Jenkins Credential 설정
동일
3. Kubernetes Secret 설정
동일

4. Jenkins Kubernetes 설정 정보 추가
Jenkins > configuration > Cloud > Kubernetes

5. 

kubectl create secret -n default generic kube-config --from-file=$HOME/.kube/config

Jenkins > configuration > Cloud > Kubernetes
설정 추가
volumes: [secretVolume(secretName: 'kube-config', namespace: 'ns-jenkins', mountPath: '/root/.kube')])


kubectl edit clusterrolebinding/cluster-admin

subjects:
- kind: ServiceAccount
  name: default
  namespace: default

6. 
```sh
def remote = [:]
remote.name = 'test'
remote.host = '192.168.10.77'
remote.user = 'actmember'
remote.password = 'jeep8walrus'
remote.allowAnyHosts = true

pipeline {
  agent {label 'jenkins-template'}
  
  environment {
    registry = "docker.sds-act.com/eureka-test"
    dockerImage = ''
    buildnum = ''
  }
  
  stages {
  
    stage('Cloning Git') {
      steps {
        git credentialsId: 'min0418', url: 'https://github.com/SDSACT/coe-eureka.git'
      }
    }   
       
    stage('Build Project') {
      steps {
        container('maven') {
            script{
                sh "mvn clean install -Dprofile=kube -DskipTests=true"
            }
        }
      }
    }
    stage('Building image') {
      steps{
        container('docker') {
            script {
              buildnum = '${BUILD_NUMBER}'
              dockerImage = docker.build registry + ":" + buildnum
            }
        }
      }
    }
    stage('Deploy Image') {
      steps{
        container('docker') {        
            script {
              docker.withRegistry('https://docker.sds-act.com', 'dockeruser' ) {
                dockerImage.push()
              }
            }
        }
      }
    }
    stage('Kube Deploy') {
      steps{
        container('kubectl') {   
          script {
            sh ('kubectl run leo-test --env="SPRING_PROFILES_ACTIVE=kube" --image=docker.sds-act.com/eureka-test:' + buildnum)
          }
        }
      }
    }
  }
}
```

** 추가
- admin Group에 defalut:defalut 추가하기
- env (profile설정 안했는데 kube로 시작됨)
- 이미지 넣어서 가이드 보충
- User부터 다시 만들기 
http://bryan.wiki/296?category=261316

