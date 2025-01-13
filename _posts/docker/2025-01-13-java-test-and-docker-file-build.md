---
title: "Dockerfile로 Java 프로젝트를 테스트하고 빌드하기"

toc: true
toc_sticky: true

categories:
    - docker
tags:
    - Java
    - Docker
    - Dockerfile
---


# Dockerfile로 Java 테스트 후 빌드하기

이번에 개발한 프로젝트에서는 Docker Image를 활용하여 AWS EC2에 자동으로 배포하는 CI/CD를 구성해 두었다. 지금은 개선했지만, 당시에는 테스트 자동화가 이루어지지 않아 배포 전에 Dockerfile을 통해 테스트를 함께 진행했다. 이미 작성했던 Dockerfile인 만큼 해당 방식도 기록을 남겨보고자 한다.


## Dockerfile 만들기

다음의 Dockerfile은 Multi-stage build를 사용한다. 이를 통해 테스트 환경과 프로덕션 환경을 분리한다.
```dockerfile
# Build Image
FROM amazoncorretto:21-alpine AS TEMP_BUILD_IMAGE
ENV APP_HOME=/app
WORKDIR $APP_HOME

COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

COPY src ./src

RUN ./gradlew build --stacktrace

# App 구동
FROM amazoncorretto:21-alpine

ENV ARTIFACT_NAME=timely-0.0.1-SNAPSHOT.jar
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY --from=TEMP_BUILD_IMAGE $APP_HOME/build/libs/$ARTIFACT_NAME $APP_HOME/app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
```

위에서부터 차례대로 설명한다.

```dockerfile
FROM amazoncorretto:21-alpine AS TEMP_BUILD_IMAGE
```
Amazon Corretto JDK 21 Alpine 이미지를 기반으로 첫 번째 빌드 스테이지를 생성한다. 후에 빌드한 `jar` 파일을 가져오기 쉽도록 `TEMP_BUILD_IMAGE`라는 별칭을 달아준다.

```dockerfile
ENV APP_HOME=/app
WORKDIR $APP_HOME
```
`APP_HOME`라는 이름으로 `/app` 위치를 환경변수로 추가한다. 그리고 작업 디렉터리를 `/app`으로 설정한다.

```dockerfile
COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

COPY src ./src
```
이제 Java 프로젝트를 빌드하기 위해 필요한 파일들을 복사한다.

```dockerfile
RUN ./gradlew build --stacktrace
```
이제 Gradle을 사용하여 애플리케이션을 빌드한다. 여기서 test도 함께 진행된다.

만약 테스트를 제외하고 싶다면 다음과 같이 수정하면 된다.
```dockerfile
RUN ./gradlew build -x test
```

이제 테스트 과정은 완료되었다. 만약 프로젝트가 테스트를 통과하지 못할 경우 오류가 발생하여 다음 단계로 넘어가지 못하게 된다.

이제 다음 스테이지로 넘어가서 프로덕션 환경으로 이동한다.

```dockerfile
FROM amazoncorretto:21-alpine
```
위와 같이 Amazon Corretto JDK 21 Alpine 이미지를 기반으로 두 번째 스테이지를 시작한다.

```dockerfile
ENV ARTIFACT_NAME=spring-boot-app-0.0.1-SNAPSHOT.jar
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY --from=TEMP_BUILD_IMAGE $APP_HOME/build/libs/$ARTIFACT_NAME $APP_HOME/app.jar
```

`ENV ARTIFACT_NAME`를 통해 빌드될 애플리케이션의 이름을 설정해준다.
그리고 이전에 한 것과 마찬가지로 `APP_HOME` 환경 변수를 등록하고, 작업 디렉터리를 설정한다.

그리고, 이전 스테이지인 TEMP_BUILD_IMAGE에서 빌드된 `jar` 파일을 현재의 스테이지로 복사해 온다.

이렇게 되면 test 혹은 build를 위해 복사했던 프로젝트 파일들을 포함하지 않고 Docker Image를 빌드할 수 있다.

```dockerfile
EXPOSE 8080
```
컨테이너가 8080 포트를 사용함을 명시한다.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar", "--spring.profiles.active=prod"]
```
컨테이너가 시작될 때 실행할 명령어를 설정한다.

## 마치며
이제는 배포 브랜치에 Pull Request를 생성할  때는 Test가 자동으로 이루어지기 때문에 Dockerfile에서 Test를 함께 진행할 필요는 없어졌다. 그럼에도 Multi-stage build 방식을 사용할 경우 Docker Image의 용량을 줄일 수 있기 때문에 적극 활용하고자 한다.