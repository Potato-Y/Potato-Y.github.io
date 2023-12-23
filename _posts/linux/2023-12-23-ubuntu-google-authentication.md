---
title: "Ubuntu Google 2차 인증 추가하기"

toc: true
toc_sticky: true

categories:
    - Linux
tags:
    - Linux
    - Security
    - Ubuntu
---

# Ubuntu Google Authenticator

## SSH
### 설치

Ubuntu에 Google authenticator를 설치한다.

```bash
> sudo apt-get install libpam-google-authenticator
```

스마트폰에 맞는 스토어에 접속하여 Google authenticator를 설치한다.

### 인증키 발급

터미널에 다음의 명령어를 입력한다. `sudo`를 사용하면 `root`의 2차 인증 키가 발급되므로 빼고 실행한다.

```bash
> google-authenticator
```

#### 인증키 설정

```bash
Do you want authentication tokens to be time-based (y/n) y
```
-> 시간 기반의 토큰을 사용할 것인지 질문한다. 시간 기준/카운터 중에 하나를 선택할 수 있다.

이후 QR 코드가 표시된다. `Google Authenticator` 앱에서 `QR 코드 스캔` 버튼을 통해 카메라를 실행시켜 인식시킨다.

```bash
Your new secret key is: ...
Enter code from app (-1 to skip): 
```
-> 위 작업을 통해 추가된 2차 인증 키를 입력한다. 만약 건너뛰려면 `-1`를 입력하면 된다.

```bash
Your emergency scratch codes are:
  691...
  205...
  154...
  191...
  312...
```
-> 2차 인증에 실패했을 때 사용할 수 있는 코드다. 안전하게 보관해야 하며, 일회성 코드이기 때문에 가능하면 사용한 뒤에 2차 인증 키를 새로 발급한다.

```bash
Do you want me to update your "/home/user_name/.google_authenticator" file? (y/n)  y
```
-> 위의 내용을 저장할 것인지 질문한다. 당연히 저장해야 2차 인증에 사용할 수 있다. 혹시 경로에 `/root/`가 포함되어 있다면 저장하지 않고 다른 user로 변경하여 다시 한다.

```bash
Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n)  y
```
-> 한번 사용한 코드는 재사용하지 않도록 설정한다.

```bash
By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n
```
-> 2차 인증 코드는 시간에 의존한다. 서버와 클라이언트의 시간이 동일하지 않더라도 허용할 것인지 묻는 질문이다. 보안을 위해 n를 입력한다. 

```bash
If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```
-> 로그인 시도 횟수를 제한한다. 연속해서 틀릴 경우 시간제한이 끝난 뒤에 다시 시도할 수 있다.

### SSH PAM 설정

구글 인증을 위해 PAM 파일을 수정한다.
`/etc/pam.d/sshd`
```bash
auth required pam_google_authenticator.so
```
맨 아래에 위의 내용을 추가한다. 만약 `nullok`를 사용하면 2차 인증이 선택적이기 때문에 제거하는 것이 좋다.

### SSHD 설정 변경

`/etc/ssh/sshd_config`
```bash
UsePAM yes
AuthenticationMethods publickey,keyboard-interactive
```

- `publickey`: ssh key
- `keyboard-interactive`: 2차 인증

#### CentOS 7
`/etc/ssh/sshd_config`
```bash
# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
ChallengeResponseAuthentication yes
```

#### Ubuntu
`/etc/ssh/sshd_config`
```bash
# Change to yes to enable challenge-response passwords (beware issues with
# some PAM modules and threads)
KbdInteractiveAuthentication yes
```

### 서비스 재실행
```bash
systemctl restart sshd.service or ssh.service
```

이렇게 하고 `ssh` 접속을 하면 구글 2차 인증을 추가로 질문한다. 참고로 Termius에서 2차 인증을 사용하려면 유료 버전을 사용해야 한다. 또한 Filezilla에서 2차인증을 통해 SFTP를 사용하려면 유료 버전을 사용해야 하는 것으로 보인다.

### 추가 설정
위와 같이 설정하면 계정 패스워드도 함께 질문한다. 만약 유저의 패스워드는 묻지 않으려면 다음과 같은 수정을 한다.

`/etc/pam.d/sshd`
```bash
# Standard Un*x authentication.
# @include common-auth
```
위의 내용을 주석 처리하면 된다.

## SU에 적용

계정을 변경할 때에도 2차 인증을 사용할 수 있다.

`/etc/pam.d/su`
```bash
auth       [success=1 default=ignore] pam_succeed_if.so user = root
auth       required     pam_google_authenticator.so
```
위 내용은 `root`를 제외하고 모든 유저로 계정을 변경할 때 2차 인증을 묻도록 하는 설정이다.

만약 root를 포함한 모두에게 적용하려면 아래와 같이 설정하면 된다.

`/etc/pam.d/su`
```bash
auth required pam_google_authenticator.so
```

---
## 참고 자료

[How To Set Up Multi-Factor Authentication for SSH on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-multi-factor-authentication-for-ssh-on-ubuntu-16-04)

[Ubuntu SSH Key & MF2 (Google Authentication 이용) 구현해보기](https://blog.naver.com/PostView.naver?blogId=happy_jhyo&logNo=223024752148&redirect=Dlog&widgetTypeCall=true&directAccess=false)

