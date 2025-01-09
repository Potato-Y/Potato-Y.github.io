---
title: "GitHub Pull Request 시 Actions으로 Java 코드 테스트 성공하고 Merge 하기"

toc: true
toc_sticky: true

categories:
    - ci
tags:
    - GitHub
    - Actions
---

# 테스트 성공 시 merge 할 수 있도록 하기

프로젝트를 진행하며 병합하기 전에 수작업으로 전체 테스트를 진행해왔지만, 생각보다 테스트 하는 것을 놓치는 경우가 많이 생겼다. 역시 수작업으로 매번 테스트를 하는 것은 휴먼 에러가 발생하기 쉽고, 앞으로를 생각하여 테스트가 성공해야지만 Merge 버튼이 활성화 되도록 설정을 해보고자 한다.

필자는 예제를 위한 java 프로젝트를 생성하였다. [예시 프로젝트](https://github.com/Potato-Y/Actions_Pull_Request_Test)를 기반으로 설명한다.

## workflows 생성

다음 `path`를 참고하여 GitHub 저장소에 파일을 생성한다.

`.github/workflows/ci_test.yml`
```yml
name: CI Pipeline

on:
  pull_request:
    branches: [ main, develop ] # 해당 브랜치에 pull request가 생성될 때 실행

permissions: write-all # workflow가 모든 쓰기 권한을 갖는다.

jobs:
  build:
    runs-on: ubuntu-latest # Ubuntu 최신 버전에서 실행

    steps:
    - name: Checkout code
      uses: actions/checkout@v4 # Repository의 코드를 가져오기

    - name: Set up JDK 21 
      uses: actions/setup-java@v4 # Java 21 버전 설치
      with:
        distribution: 'temurin'
        java-version: '21'

    - name: Grant execute permission for gradlew
      run: chmod +x gradlew # Gradle wrapper 스크립트에 실행 권한 부여

    - name: Test with Gradle
      run: ./gradlew --info test # 프로젝트의 테스트 실행

    - name: Publish Test Results
      uses: EnricoMi/publish-unit-test-result-action@v2 # 테스트 결과 출력
      if: always() # 성공, 실패 여부와 상관 없이 실행
      with:
        junit_files: '**/build/test-results/test/TEST-*.xml'
```

각 항목에 대한 내용은 주석을 참고하면 된다.

## actions 동작 확인하기

### 브랜치 생성 및 코드 작성

이제 `develop`와 `feature/sum` 브랜치를 차례대로 생성한다. 그리고 `feature/sum`에 더하기 메서드를 추가하고 Pull Request를 생성한다.

Pull Request를 생성하고 잠시 기다리면, 다음 사진과 같이 위에서 정의한 작업이 실행되는 것을 볼 수 있다.

![image](https://github.com/user-attachments/assets/1541affb-95c5-415c-8ce8-a3e88586b9d7)

만약 테스트에 성공하면 다음 사진과 같이 결과 내용이 담긴 comment가 달리게 된다.

![image](https://github.com/user-attachments/assets/ef62965a-4ce8-4b76-a228-ac6bcb2e5f5e)

[관련 pull request](https://github.com/Potato-Y/Actions_Pull_Request_Test/pull/3)

## 테스트 성공 시에만 merge 하기

테스트를 자동으로 진행하더라도, 실패 시 merge를 하면 자동으로 테스트 하는 의미가 없어진다. 테스트에 실패할 경우 merge를 할 수 없도록 Rule을 설정할 수 있다.

해당 Repository에서 `Settings > Branches > Add classic branch protection rule`으로 이동한다.

![image](https://github.com/user-attachments/assets/89dde7a7-b176-4e82-bcd2-246ed7490397)

### Rule 생성

![image](https://github.com/user-attachments/assets/271a29f8-4b38-4a73-9321-9cb796667324)

위의 사진을 참고하여 내용을 추가한다. 검색 항목에는 `Test Results`를 검색하여 표시되는 내용을 추가하면 된다.

위와 같이 설정이 완료되면 하단의 `Create` 버튼을 눌러 마무리한다. 필자의 경우에는 `main`, `develop` 브랜치에 대해 rule을 추가했다.

### 실패 시 merge 여부 확인하기

이제 실패하는 테스트를 추가해서 merge가 가능한지 확인해본다. 일단 새로운 pull request를 생성하면 아까와는 살짝 다른 화면이 출력되는 것을 확인할 수 있다.

![image](https://github.com/user-attachments/assets/7be8b95b-c666-4b35-be4d-29109185b2b2)

테스트에 통과하기 전이기 때문에 버튼이 비활성화 되어있다.

그리고 테스트가 완료되면 다음 사진과 같이 실패에 대한 코멘트가 작성되고, 버튼은 여전히 비활성화 상태인 것을 확인할 수 있다.

![image](https://github.com/user-attachments/assets/b3ecf42d-90b2-4b2c-8a56-2e0bc663cd76)

[관련 pull request](https://github.com/Potato-Y/Actions_Pull_Request_Test/pull/4)

잘못된 테스트 코드를 수정하여 다시 pull request를 생성하면, 테스트가 정상 통과하고 merge 버튼이 활성화 되는 것을 확인할 수 있다.

---

이제 로컬 환경에서 직접 테스트를 돌리는 번거로움에서 벗어날 수 있게 된다.

