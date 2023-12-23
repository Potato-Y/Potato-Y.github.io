---
title: "Ubuntu sudo 명령어 권한 관리하기 / 전체 혹은 일부"

toc: true
toc_sticky: true

categories:
    - Linux
tags:
    - Linux
    - Security
    - Ubuntu
---

# Ubuntu Sudo 권한 관리

## 특정 유저의 모든 명령어 사용

특정 사용자가 모든 명령어를 `sudo`로 사용할 수 있도록 하려면 다음과 같이 그룹을 추가한다.

```bash
> usermod -aG sudo username
```

## 특정 명령어만 허용

다음의 설정을 통해 그룹별, 혹은 유저별로 특정 명령에 대하여 sudo 사용을 허용할 수 있다.

다음의 명령어를 통해 sudo 명령어 설정에 대한 편집기를 실행한다.
```bash
visudo
```
> 위와 같이 하지 않고 파일을 직접적으로 실행할 경우 문법에 오류가 있어도 검증이 불가능하기 때문에 꼭 위의 명령어로 편집기를 열자.

### 그룹별로 지정하기

```bash
%dadogk ALL=(ALL) /usr/bin/vim, /usr/bin/vi, /usr/bin/systemctl
```

위와 같이 그룹명 앞에 `%`를 붙여준다. 그런 다음 허용할 기능에 대해 적는다. 만약 모든 명령어에 대해 허용하려면 아래와 같이 한다.

```bash
%dadogk ALL=(ALL) ALL
```

만약 특정 유저에 대해서 허용하고 싶다면 `%group_name`을 `user_name`으로 대체한다.
