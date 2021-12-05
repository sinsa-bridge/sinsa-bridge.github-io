---
layout: post
title: Docker 배포
author: 이강운
subtitle: Docker를 활용한 Springboot 어플리케이션 배포
categories: docker
tags: [docker]
---

개요
-------------

개발한 프로젝트를 Docker를 통해 배포한다.<br>
사내 프로젝트인 sunny-golf-api를 배포한다.
<br><br>

환경
-------------
> 운영체제: Mac OS M1<br>
> 자바 : Java 8<br>
> 스프링 : Springboot 2.5.x<br> 
> 빌드툴 : Gradle<br>
> 프로젝트 : docker-demo - [GIT 정보][1]<br>
> 기타: Docker Hub, Portainer - [서버정보][2]

<br>

진행
-------------
테스트용 프로젝트 docker-demo를 이용하여 빌드/배포 전 Docker Engine을 설치한다.<br>
설치 파일은 [Docker 공식사이트][3]에서 운영체제 알맞게 설치한다.<br>

docker-demo 프로젝트는 간단한 동작 확인을 위해 /로 접근 시 환영문구를 출력하는 기능이 있다. 
```java
@RestController
@SpringBootApplication
public class SinsaDockerDemoApplication {

    @RequestMapping("/")
    public String index() {
        return "Welcome to Docker Demo!";
    }
    public static void main(String[] args) {
        SpringApplication.run(SinsaDockerDemoApplication.class, args);
    }

}
```

이후 Spring을 기동시켜 정상 동작하는 지 확인한다.<br>

<br>

***Step.01&nbsp; Dockerfile 작성***<br>
```docker
# Docker 컨테이너 JDK 8 이미지를 기반으로 동작시킨다.
FROM openjdk:8-jdk-alpine
# Docker 컨테이너에서 사용할 포트를 개방시킨다
EXPOSE 8080

# Docker 컨테이너에서 사용할 group, user를 지정
RUN addgroup -S sinsa
RUN adduser -S sinsa -G sinsa

# Docker 컨테이너를 실행하는 user:group을 지정
USER sinsa:sinsa
# * 이때 컨테이너의 사용자/그룹의 uid, gid는 호스트(실제 물리서버)와 공유되니 유의

# TimeZone지정 Docker 컨테이너는 기본적으로 UTC로 설정되기 때문에 TimeZone을 서울로 지정
ENV TZ="Asia/Seoul"

# 배포하고자하는 JAR 지정
ARG JAR_FILE=build/libs/docker-demo-0.0.1-SNAPSHOT.jar
ADD ${JAR_FILE} docker-demo.jar

# JAVA 어플리케이션 실행 시 필요한 JVM옵션을 추가
ENTRYPOINT ["java", "-jar", "-Dfile.encoding=UTF-8", "/docker-demo.jar"]
```

프로젝트 최상단 디렉토리에 파일명 'Dockerfile'을 추가한다.
<br><br>

***Step.02&nbsp; Gradle 빌드***<br>

```gradle
./gradlew clean build
```
```maven
./mavenw clean install   //메이븐의 경우
```

<br>

***Step.03&nbsp; Docker 이미지 생성***<br>

Docker Hub 계정에 로그인한다. 회사 Docker Hub 정보는 - [서버정보][2]에서 참고하길 바란다.<br>
로컬에서만 테스트하고자 하면 Docker Hub에 로그인할 필요는 없다.
```cmd
docker login -u [계정정보]
password: [비밀번호]
```
<br>
Docker Hub에 로그인 후 Docker Image를 빌드한다. Docker 빌드 명령어는 반드시 Dockerfile이 위치한 경로에서 진행한다.<br>


이때 M1사용자와 x64 프로세서 기반 운영체제 사용자와 build시 플랫폼을 다르게 지정해줘야하는데 우리 회사 서버의 경우 linux amd x64 기반이기 때문에 M1 사용자의 경우 플랫폼 호환이 되지 않아 Docker 동작시 오류가 발생한다.<br>

하지만 Docker에서는 buildx라는 멀티 플랫폼을 지원하는 툴이 있기 때문에 [관련 사이트][3] 에서 buildx를 확인하여 설치한다. <br>


```cmd 
docker buildx build --platform linux/amd64 --load --tag sinsabridge/docker-demo:v0.1 . // M1
```

```cmd 
docker build --tag sinsabridge/docker-demo:v0.1 . // x64 
```
<br>

build 후 이미지 생성 확인
```cmd
deneb@igang-un-ui-MacBookPro ~/d/p/docker-demo> docker images
REPOSITORY                   TAG       IMAGE ID       CREATED          SIZE
sinsabridge/docker-demo      v0.1      37ac4f95439c   51 seconds ago   122MB
```

<br>

***Step.04&nbsp; Docker Hub 이미지 올리기***<br>

생성된 이미지를 Docker hub에 push 한다. 이때 Docker hub에 로그인되어 있어야 한다<br>
```cmd
deneb@igang-un-ui-MacBookPro ~/d/p/docker-demo> docker push sinsabridge/docker-demo:v0.1
The push refers to repository [docker.io/sinsabridge/docker-demo]
ca7aa367e7c9: Pushing [=====================>                             ]  7.472MB/17.5MB

```

docker hub페이지에서 docker image를 확인한다.<br>

![docker hub 확인](/assets/images/posts/docker/docker_01.png)

<br>

***Step.05&nbsp; 서버에서 Docker 이미지 동작 시키기***<br>

사내 sina1 서버에서 docker image pull을 진행한다.<br>

```shell
sinsa@sinsabridge-1:/home$ docker pull sinsabridge/docker-demo:v0.1
v0.1: Pulling from sinsabridge/docker-demo
e7c96db7181b: Already exists 
f910a506b6cb: Already exists 
c2274a1a0e27: Already exists 
1b3e4ca8091d: Pull complete 
cddad02b3621: Pull complete 
61ea1b9ea53a: Pull complete 
Digest: sha256:049d58b210b75eea807f548a241b463a5c4b35f85e1f24efe0fd5bdb40779b50
Status: Downloaded newer image for sinsabridge/docker-demo:v0.1
docker.io/sinsabridge/docker-demo:v0.1

```

이미지를 확인한다.<br>
```shell
sinsa@sinsabridge-1:/home$ docker images | grep docker-demo
sinsabridge/docker-demo              v0.1      37ac4f95439c   26 minutes ago   122MB
```
<br>

이제 Docker 이미지를 Run 하면 되는데 아래의 명령어로 실행하겠다.

docker run -d --name docker-demo -it -p 4000:8080sinsabridge/docker-demo:v0.1

-d : detached 옵션, 컨테이너를 백그라운드에서 실행시킨다. 서버에서 직접 실행하는 경우 -d 옵션을 주지 않으면
docker를 실행한 세션은 docker 세션에서 빠져나오는 순간 해당 컨테이너도 종료된다.<br>

--name : 컨테이너의 식별 name을 정의한다. 별도로 지정하지 않으면 docker에서 임의의 값을 지정<br>

-it : <br>

-p [host port]:[container port] : 호스트에서 동작하는 포트번호와 컨테이너 내부에서 동작하는 포트를 지정한다. 현재 데모 프로젝트의 경우 8080으로 동작하고 외부에서는 4000번으로 유입되게끔 설정하였기 때문에
-p 4000:8080으로 설정하였다. 이러한 행위는 보통 포트포워딩이라고 한다.<br>

```shell
sinsa@sinsabridge-1:/home$ docker run -d --name docker-demo -it -p 4000:8080sinsabridge/docker-demo:v0.1
f543f65264e0a154622e083e52df677e1d58064ba76035b956ae81ca0f941811
sinsa@sinsabridge-1:/home$ docker ps | grep docker-demo
f543f65264e0   sinsabridge/docker-demo:v0.1          "java -jar -Dfile.en…"   7 seconds ago   Up 6 seconds          0.0.0.0:4000->8080/tcp, :::4000->8080/tcp                                                                                              docker-demo
```


***Step.06&nbsp; 확인***<br>

![docker hub 확인](/assets/images/posts/docker/docker_02.png)

<br>

마치며
-------------
개발한 프로젝트를 통해 Docker 이미지를 생성해 Docker 서버에 배포하였다.<br>
본 문서는 기본적인 Docker 배포 방식만 설명하였기에 추가적인 설명은 다른 문서에서 진행하겠다.<br>

<br>


[1]: https://github.com/sinsa-bridge/sunny-golf-api

[2]: https://docs.google.com/spreadsheets/d/1LoUkgDEFxzyPy2gPzXG0IDIT0vRcqf-rSvPzUhbWWWY/edit#gid=0

[3]: https://docs.docker.com/engine/install/

[4]: https://docs.docker.com/buildx/working-with-buildx/