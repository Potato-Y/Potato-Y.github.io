---
title: "Mac jenv로 자바 버전 관리하기"

toc: true
toc_sticky: true

categories:
    - dev_env
tags:
    - Mac
    - Java
---

# jenv로 Java 버전 관리하기

`jenv`는 `Python`의 `pyenv`, `Node`의 `nvm`과 유사한 역할을 한다.

## Java 설치

사용하고자 하는 버전의 `jdk`를 설치한다.

설치 가능한 `jdk`는 다음의 명령어로 확인할 수 있다.
```bash
> brew search jdk
==> Formulae
openjdk             openjdk@17          jd                  cdk
openjdk@11          openjdk@8           mdk

==> Casks
adoptopenjdk               jdk-mission-control        oracle-jdk-javadoc
gama-jdk                   microsoft-openjdk          sapmachine-jdk
graalvm-jdk                oracle-jdk                 semeru-jdk-open
```

필자의 경우에는 `openjdk@17`, `openjdk@11`를 설치했다. 만약 8 버전이 필요하다면 다음의 링크에서 jdk 파일을 다운받자.

> https://www.azul.com/downloads/?version=java-8-lts&os=macos&architecture=arm-64-bit&package=jdk#zulu

하단으로 이동하면 `ARM 64-bit` 파일을 다운로드 할 수 있다.

## jenv

### jenv 설치

```bash
brew install jenv
```
brew를 통해 `jenv`를 설치한다.

설치가 완료되면 아래의 명령어를 통해 `PATH`를 추가하고 적용한다.
```bash
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc

source ~/.zshrc
```

### jenv 설정

`jenv`에 설치한 `jdk`를 추가한다.

```bash
jenv add /opt/homebrew/opt/openjdk@17
jenv add /opt/homebrew/opt/openjdk@11
jenv add /Users/-/dev/jdk/zulu8.74.0.17-ca-jdk8.0.392-macosx_aarch64 # 필자가 따로 설치한 jdk 8
```

다음의 명령어를 통해 연결한 `jdk`를 확인할 수 있다.
```bash
> jenv versions
* system (set by /Users/potatoy/.jenv/version)
  1.8
  1.8.0.392
  11.0
  11.0.21
  17.0
  17.0.9
  openjdk64-11.0.21
  openjdk64-17.0.9
  zulu64-1.8.0.392
```

### jenv jdk 버전 설정

#### 전역 설정

다음의 명령어로 원하는 버전을 지정한다.
```bash
jenv global openjdk64-17.0.9
```

터미널을 다시 시작하면 다음과 같이 적용된 것을 확인할 수 있다.

```bash
> java -version
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment Homebrew (build 17.0.9+0)
OpenJDK 64-Bit Server VM Homebrew (build 17.0.9+0, mixed mode, sharing)
```

#### 특정 프로젝트 설정

특정 프로젝트와 같은 디렉토리에만 적용하려면 다음의 명령어를 사용하면 된다.

```bash
jenv local openjdk64-11.0.21
```

### jenv에서 특정 jdk 제거

특정 `jdk`를 제거하려면 다음의 명령어를 사용하면 된다.
```bash
jenv remove <name>
```