## Dockerizing
수정된 프로젝트를 빌드하여 도커 이미지로 만든 후 레지스트리에 Push 합니다.

### Dockerfile 생성
프로젝트 경로에 Dockerfile을 작성합니다.
```
FROM openjdk:8-jre-alpine
ENV APP_FILE content-service-1.1.0-SNAPSHOT.jar
ENV APP_HOME /usr/app
COPY build/libs/$APP_FILE $APP_HOME/
WORKDIR $APP_HOME
ENTRYPOINT ["sh", "-c"]
CMD ["exec java -jar $APP_FILE"]
```
### Image Build
프로젝트를 빌드하여 도커파일에서 설정한 경로에 jar 파일이 만들어지면 아래 명령어로 도커 이미지를 생성합니다.
```sh
$ docker build -t docker.sds-act.com/contents .
```
### Image Build 결과 확인
```sh
➜  coe-kubernetes git:(master) ✗ docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
docker.sds-act.com/contents   latest              680e303d2338        1 days ago          205MB
```
### Image Push

```sh
# 현재 예제에서는 Nexus를 Docker Registry로 활용하였습니다. 
$ docker push docker.sds-act.com/contents
```
