---
title: "Spring boot + MariaDB + Nginx docker-compose 만들기"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - Docker
---

# Spring Boot - Dockerfile 만들기

## profiles.active 사용하기

해당 방법은 MariaDB에 접속하기 위한 정보를 `application-prod.yml`에 저장하여 이를 통해 Spring Boot를 구동할 때 사용할 수 있는 방법 중 하나이다.

우선 `Dockerfile`의 전체 내용은 다음과 같다.

`Dockerfile`
```Dockerfile
# Build Image
FROM openjdk:17-jdk-alpine AS TEMP_BUILD_IMAGE

ENV APP_HOME=/app
WORKDIR $APP_HOME

COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

COPY src ./src

RUN chmod +x ./gradlew
RUN ./gradlew build -x test --stacktrace

# Production Image
FROM openjdk:17-jdk-alpine

ENV ARTIFACT_NAME=dadogk-0.0.1-SNAPSHOT.jar
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY --from=TEMP_BUILD_IMAGE $APP_HOME/build/libs/$ARTIFACT_NAME $APP_HOME/app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "app.jar"]
```

위의 `Dockerfile`은 크게 두 영역으로 나누어 볼 수 있다. 

- `Build Image`: 프로젝트를 복사하고, 빌드한다.
- `Production Image`: 빌드한 프로젝트를 격리된 공간에서 실행한다.

### Build Image

`Build Image` 영역을 위에서부터 차례대로 본다.

```Dockerfile
FROM openjdk:17-jdk-alpine AS TEMP_BUILD_IMAGE
```

`openjdk:17-jdk-alpine` 이미지를 사용하고 `TEMP_BUILD_IMAGE`라는 이름을 붙인다.

```Dockerfile
ENV APP_HOME=/app
WORKDIR $APP_HOME
```

`APP_HOME`이라는 변수를 생성하고 기본 디렉터리를 저장한다. 사실상 현재 수준의 Build Image 단계에서는 변수 없이 한 줄로 작성해도 무관하다.

```Dockerfile
COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

COPY src ./src
```

프로젝트의 복사는 자주 변경되는 파일을 개별적으로 복사한다. 의존성과 같이 자주 변경되지 않는 부분은 캐싱을 통해 빠르게 빌드될 수 있도록 한다.

```Dockerfile
RUN chmod +x ./gradlew
RUN ./gradlew build -x test --stacktrace
```

`gradlew`를 실행할 수 있도록 권한을 주고, test 코드를 제외하고 빌드한다. 권한을 주는 작업을 제거해도 문제가 발생하지 않았지만, 혹시 모를 상황에 대비하여 포함시켰다.

### Production Image

이제 위의 단계에서 빌드된 `.jar` 파일을 사용할 차례다.

```Dockerfile
WORKDIR $APP_HOME
COPY --from=TEMP_BUILD_IMAGE $APP_HOME/build/libs/$ARTIFACT_NAME $APP_HOME/app.jar
```

디렉터리를 이동하고, `--from` 옵션을 사용하여 `TEMP_BUILD_IMAGE` 이미지에서 생성한 `dadogk-0.0.1-SNAPSHOT.jar` 바이너리 파일을 해당 이미지로 복사해온다.

```Dockerfile
EXPOSE 8080
```

컨테이너가 사용할 포트를 지정한다.

```Dockerfile
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "app.jar"]
```

`ENTRYPOINT`는 컨테이너가 시작될 때 필수적으로 특정 명령을 실행해야 하는 경우에 적합하다. 위의 명령어를 통해 컨테이너가 시작될 때 `jar` 파일을 `prod` 프로필로 실행시킨다.

> 참고
>- [ENTRYPOINT, CMD 차이](https://velog.io/@inhalin/ENTRYPOINT-CMD-%EC%B0%A8%EC%9D%B4)
>- [Dockerfile Multi-stage build](https://kimjingo.tistory.com/63)

## ENV 이용하기

만약 profiles를 지정하지 않고, 직접 Dockerfile에 환경 변수를 추가하고 싶다면 다음과 같이 추가해도 방법을 사용해도 된다.

`Dockerfile 예시`
```Dockerfile
...

# Production Image
FROM openjdk:17-jdk-alpine

ENV ARTIFACT_NAME=dadogk-0.0.1-SNAPSHOT.jar
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY --from=TEMP_BUILD_IMAGE $APP_HOME/build/libs/$ARTIFACT_NAME $APP_HOME/app.jar

ENV SPRING_DATASOURCE_USERNAME=dadogk_service
ENV SPRING_DATASOURCE_PASSWORD=wjsansrk

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

필자의 경우에는 `docker-compose.yml` 파일에 필요한 내용들을 추가했다.

# docker-compose 만들기

`docker-compose`는 단일 서버에서 여러 개의 컨테이너를 하나의 서비스로 정의하여 컨테이너의 묶음으로 관리할 수 있는 작업 환경을 제공하는 관리 도구이다. 이를 통해 Spring Boot 프로젝트에 필요한 서비스를 함께 운영 서버와 동일하게 구축할 수 있다.

profile을 사용할 때의 기준으로 전체 코드는 아래와 같다.

`docker-compose.yml`
```yml
version: '3'

services:
  maria-db:
    image: mariadb:11.4
    container_name: dadogk_db
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=dadogk
      - MYSQL_USER=user
      - MYSQL_PASSWORD=pwd
      - TZ=Asia/Seoul
    volumes:
      - ../db:/var/lib/mysql
    command:
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
    networks:
      - dadogk_network

  spring-boot:
    build: .
    container_name: dadogk_spring_boot_app
    depends_on:
      - maria-db
      - smtp-mailhog
    networks:
      - dadogk_network

  nginx:
    image: nginx:1.27.0
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./src/main/resources/init/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - spring-boot
    networks:
      - dadogk_network

networks:
  dadogk_network:
    driver: bridge
```

## Maria DB

```yml
  maria-db:
    image: mariadb:11.4 # 이미지 지정
    container_name: dadogk_db # 컨테이너 이름 지정
    ports:
      - "3306:3306" # 포트 Open
    environment: # 기본적인 설정 값들을 환경 변수로 지정
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=dadogk
      - MYSQL_USER=user
      - MYSQL_PASSWORD=pwd
      - TZ=Asia/Seoul
    volumes: # 저장소를 이어준다. 이를 통해 컨테이너를 재시작하여도 DB 내용이 저장된다.
      - ../db:/var/lib/mysql # :을 중심으로 왼쪽은 local, 오른쪽은 컨테이너 위치이다.
    command: # MariaDB 서버의 문자 집합 및 정렬 순서를 UTF-8로 설정한다.
      - '--character-set-server=utf8mb4'
      - '--collation-server=utf8mb4_unicode_ci'
    networks: # dadogk_network 네트워크에 연결한다.
      - dadogk_network
```

여기서 지정한 컨테이너 이름은 Spring Boot에서 DB에 접속하기 위해 필요한 주소로 쓰인다. 위 내용을 기준으로는 DB 접속 주소가 `jdbc:mariadb://dadogk_db:3306/dadogk`가 된다.

처음에 이 부분 때문에 삽질을 많이 했다. 기존에는 `localhost:3306`로 했는데, 이러면 Spring 컨테이너 안에서의 `3306` 포트를 찾아서 문제가 된다. 때문에 컨테이너 이름을 통해 접속해 주면 된다.

## Spring Boot

위에서 만든 `Dockerfile`를 사용하여 컨테이너를 생성한다.

```yml
  spring-boot:
    build: . # 해당 위치에 있는 Dockerfile을 빌드한다.
    container_name: dadogk_spring_boot_app # 컨테이너 이름 지정
    depends_on: # 해당 컨테이너가 먼저 시작되어야 한다.
      - maria-db
      - smtp-mailhog
    networks: # dadogk_network 네트워크에 연결한다.
      - dadogk_network
```

여기에선 Nginx를 통해 80 포트로 연결해 줄 것이기 때문에 따로 포트를 열지 않았다.

### 환경 변수 추가

만약 profile을 지정하지 않고 환경 변수를 직접 추가한다면 아래와 같이 사용할 수도 있다.

```yml
  spring-boot:
    build: .
    container_name: dadogk_spring_boot_app
    depends_on:
      - maria-db
      - smtp-mailhog
    environment:
      - SPRING_DATASOURCE_DRIVER-CLASS-NAME=org.mariadb.jdbc.Driver
      - SPRING_DATASOURCE_URL=jdbc:mariadb://dadogk_db:3306/dadogk
      - SPRING_DATASOURCE_USERNAME=user
      - SPRING_DATASOURCE_PASSWORD=pwd

      - SPRING_MAIL_HOST=dadogk_smtp
      - SPRING_MAIL_PORT=1025
      - SPRING_MAIL_PROPERTIES_MAIL_DEBUG=false
      - SPRING_MAIL_PROPERTIES_MAIL_SMTP_STARTTLS_ENABLE=false
      - SPRING_MAIL_PROPERTIES_MAIL_SMTP_AUTH=false

      - JWT_ISSUER=???
      - JWT_SECRET_KEY=???
    networks:
      - dadogk_network
```

여기서 중요한 정보는 꼭 분리하여 저장하도록 한다.

## Nginx

```yml
  nginx:
    image: nginx:1.27.0 # 이미지 지정
    container_name: nginx # 컨테이너 이름 지정
    ports:
      - "80:80" # 포트 Open
    volumes: # 설정 파일을 연결해준다. 
      - ./src/main/resources/init/nginx.conf:/etc/nginx/nginx.conf
    depends_on: # 해당 컨테이너가 먼저 시작되어야 한다.
      - spring-boot
    networks: # dadogk_network 네트워크에 연결한다.
      - dadogk_network
```

`nginx.conf`
```conf
events { }

http {
    upstream backend {
        server dadogk_spring_boot_app:8080; # 컨테이너 이름과 포트 번호를 지정한다.
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend; # upstream의 'backend'를 넣어준다.
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 86400s;
        }
    }
}
```

## networks

```yml
networks: # 해당 네트워크를 생성한다. 
  dadogk_network:
    driver: bridge
```

이제 docker-compose를 통해 다른 사람도 쉽게 운영 서버와 동일한 환경을 구축할 수 있다.


> 참고
>- [Docker 이해하기 -4 : Docker Compose 이해하고 구성하기](https://adjh54.tistory.com/503)
>- [Docker compose로 Spring, Mysql 배포환경 구성하기](https://velog.io/@eoveol/Docker-compose%EB%A1%9C-Spring-Mysql-%EB%B0%B0%ED%8F%AC%ED%99%98%EA%B2%BD-%EA%B5%AC%EC%84%B1%ED%95%98%EA%B8%B0#2-dockerfile-%EC%9E%91%EC%84%B1)
>- [Docker compose의 네트워크부터 실습까지](https://velog.io/@hyeongjun-hub/Docker-compose%EC%9D%98-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC%EB%B6%80%ED%84%B0-%EC%8B%A4%EC%8A%B5%EA%B9%8C%EC%A7%80)
>- [Docker compose로 MySQL 실행하기](https://velog.io/@gingaminga/Docker-compose%EB%A1%9C-MySQL-%EC%8B%A4%ED%96%89%ED%95%98%EA%B8%B0)