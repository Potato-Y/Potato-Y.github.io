---
title: "GitHub Actions으로 Java 코드 스타일 확인하기"

toc: true
toc_sticky: true

categories:
    - ci_cd
tags:
    - GitHub
    - Actions
---

Flutter 혹은 Node.js 프로젝트를 빌드 하거나 실행할 때 코드를 자동으로 정렬해 주는 기능을 쉽게 사용할 수 있지만, Java 프로젝트를 할 때에는 쉽지 않았다. 그러다 찾은 것이 `spotless`이다.

# GitHub Actions를 통해 코드 스타일 확인

## Java 프로젝트 설정

### build.gradle
```gradle
plugins {
   ...
   id "com.diffplug.spotless" version "6.25.0"
}

spotless {
    java {
        // 사용하지 않는 import 제거
        removeUnusedImports()
        // 구글 자바 포맷 적용
        googleJavaFormat()

        // indentWithTabs(2) // TODO: 추후 추가 고려
        indentWithSpaces(2)

        // 공백 제거
        trimTrailingWhitespace()
        // 끝부분 New Line 처리
        endWithNewline()
    }
}
```

[intellij-java-google-style.xml](https://github.com/google/styleguide/blob/gh-pages/intellij-java-google-style.xml)과는 조금 다르게 적용된다. 그리고 세부적으로 설정할 수 있는 폭이 적다. 때문에 코드 컨벤션을 변경한지 얼마 되지 않았지만, 또 한 번 코드 컨벤션을 바꾸는 과정을 거치게 되었다.

#### 작동 확인
```bash
> ./gradlew spotlessCheck
Starting a Gradle Daemon (subsequent builds will be faster)

BUILD SUCCESSFUL in 7s
3 actionable tasks: 3 executed
```
위 명령어를 통해 정상적으로 실행되고, 코드 스타일을 지키고 있는지 확인한다. 만약 통과하지 못한다면 다음 명령어를 통해 코드 스타일을 수정시킬 수 있다.

```bash
./gradlew spotlessApply
```

## GitHub Actions 설정
GitHub 저장소에서 새로운 workflow를 생성한다.

그리고 다음과 같이 설정 파일을 작성한다.
```yml
name: Java CI with Gradle

on:
  push:
    branches: [ "develop" ]
  pull_request:
    branches: [ "develop" ]

jobs:
  code-style:

    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v3

    - name: Code style check with Google Java Style
      run: ./gradlew spotlessCheck
```

이렇게 함으로써 `gradlew` 명령어를 사용할 수 있다. `develop` 브랜치로 push, 혹은 pull request가 발생하면 `./gradlew spotlessCheck` 명령어를 실행하여 결과를 표시한다.

### 만약 Test 코드도 함께하려면
만약 이와 함께 test 코드도 함께 검사하려면 다음 과정을 추가한다

우선 GitHub 저장소에서 다음의 설정을 해야 한다.
```
Settings -> Actions -> General : Read and wirte permissions
```

그리고 workflow 파일에 다음의 내용을 추가한다.

```yml
  unit-test:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Setup Gradle
      uses: gradle/actions/setup-gradle@v3

    # Gradle test 실행
    - name: Test with Gradle
      run: ./gradlew --info test
      
    # 테스트 후 Result 출력
    - name: Publish Unit Test Results
      uses: EnricoMi/publish-unit-test-result-action@v1
      if: ${{ always() }}  # 'always' : 테스트 실패해도 Result 출력
      with:
        files: build/test-results/**/*.xml
        
      # 캐시 파일 삭제
    - name: Cleanup Gradle Cache
      if: ${{ always() }}
      run: |
        rm -f ~/.gradle/caches/modules-2/modules-2.lock
        rm -f ~/.gradle/caches/modules-2/gc.properties
```

이제 push를 하면 자동으로 test code가 실행되고, 다음과 같이 결과가 출력된다.

![image](https://github.com/Potato-Y/syiary_backend/assets/68105481/291341d4-a948-4f27-ad00-265776ceae95)

이제 test 결과가 다른 채널로도 공유되도록 설정하고, ~~코드를 고치러 가면 된다 ..~~