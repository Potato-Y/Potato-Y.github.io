---
title: "Spring Boot Gradle 버전 변경"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - Gradle
---

스프링 부트와 AWS로 혼자 구현하는 웹 서비스 책을 따라하던 중 문제가 발생했다.

```
error: variable name not initialized in the default constructor
    private final String name;
                         ^
```
이러한 오류가 발생한 것..

문제는 Gradle 버전의 차이에서 발생했다. 

>오류에 대한 내용은 https://github.com/jojoldu/freelec-springboot2-webservice/issues/78 에서 확인할 수 있다.

### Gradle 버전 다운그레이드
1. `alt + F12`로 터미널을 열어준다.
2. `./gradlew wrapper --gradle-version 4.10.2`를 입력한다.
3. `gradle/wrapper/gradle-wrapper.properties` 파일에서 버전을 확인한다.

[참고](https://velog.io/@dsunni/Spring-Boot-02-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C)

책에서 제공하는 해결 [링크](https://github.com/jojoldu/freelec-springboot2-webservice/issues/2)

### build.gradle 수정
`Gradle 5.x` 버전에서도 작동하도록 수정했던 내용을 다시 `4.x`에 맞게 수정해준다..
```js
buildscript {
    ext {
        springBootVersion = '2.1.7.RELEASE'
    }

    repositories {
        mavenCentral()
        jcenter()
    }

    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

plugins {
    id 'java'
}

group = 'io.github.potato-y'
version '1.0-SNAPSHOT'

apply plugin: 'java'
apply plugin: 'eclipse'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'

repositories {
    mavenCentral()
    jcenter()
}

dependencies {
    implementation 'org.projectlombok:lombok:1.18.24'
    compile('org.springframework.boot:spring-boot-starter-web')
    testCompile('org.springframework.boot:spring-boot-starter-test')
}

test {
    useJUnitPlatform()
}
```

처음부터 Gradle 버전을 4.x로 내리고 해야 편하다.