---
title: "ssh: rsa 방식 key로 접속하기"

toc: true
toc_sticky: true

categories:
    - Linux
tags:
    - Ubuntu
    - ssh
---

Oracle 클라우드에서 Ubuntu 인스턴스를 만들었다. 인스턴스를 만들면서 `SSH 키 추가` 항목의 옵션을 통해 `자동으로 키 쌍 생성`을 했다.
여기서 만든 키로 `ubuntu` 계정에 접속할 수 있다. 이번엔 새로 계정을 생성하였고, 그 계정에 접속할 수 있도록 `SSH Key`를 만들려고 한다.

> 만약 개인 PC에 Linux를 설치했고, 새로 SSH를 시작하려 한다면 다음의 게시글을 참고하도록 하자.
[우분투 SSH 보안 키로 접속하기](https://velog.io/@jong/%EC%9A%B0%EB%B6%84%ED%88%AC-ssh-%EB%B3%B4%EC%95%88-%ED%82%A4%EB%A1%9C-%EC%A0%91%EC%86%8D%ED%95%98%EA%B8%B0)

# SSH Key
## Server 접속
```bash
> ssh -i ssh-key.key ubuntu@ServerIP
```

비밀키와 사용자 계정 이름을 제대로 입력했다면 클라우드 서버에 접속하는 것을 성공했을 것이다.

## 계정 변경
필자는 `root` 계정에서 사용자 그룹과 계정을 새로 추가했다. GitHub 이름과 동일하게 `potatoy`로 생성했다. 다음의 명령어를 통해 사용자 계정을 변경하자.

```bash
> su potatoy
```
## Key 발급
```bash
> ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/potatoy/.ssh/id_rsa): 
Created directory '/home/potatoy/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/potatoy/.ssh/id_rsa
Your public key has been saved in /home/potatoy/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:~
```
위와 같이 표시되면 정상적으로 발급된 것이다.

다음의 위치로 이동하고, 파일이 제대로 생성되었는지 확인한다.
```bash
> cd ~/.ssh/
> ls -al
total 16
drwx------ 2 potatoy cse 4096 Nov 20 05:46 .
drwxr-x--- 4 potatoy cse 4096 Nov 20 05:46 ..
-rw------- 1 potatoy cse 2655 Nov 20 05:46 id_rsa
-rw-r--r-- 1 potatoy cse  569 Nov 20 05:46 id_rsa.pub
```

## Public Key 등록
- `id_rsa`는 ssh 버전 2 RSA 사용자 인증 정보 (authentication identity of the user)를 담고 있다. 유저 외의 다른 사람은 읽으면 안 된다.
- `id_rsa.pub`는 ssh 버전 2 RSA 퍼블릭 키 인증 정보를 담고 있다. 이 키는 `~/.ssh/authorized_keys` 이 퍼블릭 키를 이용해 로그인하려는 모든 곳의 `.ssh`에 추가되어야 한다.

[설명 출처](https://blogingming.tistory.com/entry/%EA%B3%84%EC%A0%95-%EB%B3%84-SSH-%EC%84%A4%EC%A0%95)

만든 공개키를 등록하기 위해 다음의 명령어를 통해 등록한다.

```bash
> cat id_rsa.pub>>authorized_keys
```

파일의 권한을 수정한다.
```bash
> chmod 600 authorized_keys
```

## SSH 접속
이제 생성한 key로 접속을 시도한다.

```bash
> ssh -i id_rsa potatoy@ServerIP
```

정상적으로 성공하는 것을 확인할 수 있다.

#### 더 자세한 내용
[aws-linux-instance-에-사용자별-key-pair-생성-후-접속하기](https://cloudguardians.medium.com/aws-linux-instance-%EC%97%90-%EC%82%AC%EC%9A%A9%EC%9E%90%EB%B3%84-key-pair-%EC%83%9D%EC%84%B1-%ED%9B%84-%EC%A0%91%EC%86%8D%ED%95%98%EA%B8%B0-8f9d63f51558)