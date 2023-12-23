---
title: "Spring boot SMTP 사용하기"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - SMTP
    - sendmail
    - Linux
    - Ubuntu
---

# Spring boot SMTP 사용하기

## SMTP

### sendmail 설치하기

다음의 명령어를 통해 sendmail 서비스를 설치한다.

```bash
> sudo apt-get install sendmail sendmail-cf
```

### sendmail 설정

설정을 위해 우선 sendmail 서비스를 정지한다.

```bash
> sudo systemctl stop sendmail
```

기존 설정 파일을 백업한다.

```bash
> cd /etc/mail
> sudo cp sendmail.mc sendmail.mc.backup
> sudo cp sendmail.cf sendmail.cf.backup
```

외부와 메일을 주고받기 위해 `smtp` 와 `submission`의 `DAEMON_OPTIONS`에서 Addr을 `0.0.0.0`으로 설정한다.

```bash
> sudo vim /etc/mail/sendmail.mc
```

- 기존 설정

```vim
 56 dnl DAEMON_OPTIONS(`Family=inet6, Name=MTA-v6, Port=smtp, Addr=::1')dnl
 57 DAEMON_OPTIONS(`Family=inet,  Name=MTA-v4, Port=smtp, Addr=127.0.0.1')dnl
 58 dnl DAEMON_OPTIONS(`Family=inet6, Name=MSP-v6, Port=submission, M=Ea, Addr=::1')dnl
 59 DAEMON_OPTIONS(`Family=inet,  Name=MSP-v4, Port=submission, M=Ea, Addr=127.0.0.1')dnl
```

- 수정된 설정

```vim
 56 dnl DAEMON_OPTIONS(`Family=inet6, Name=MTA-v6, Port=smtp, Addr=::1')dnl
 57 DAEMON_OPTIONS(`Family=inet,  Name=MTA-v4, Port=smtp, Addr=0.0.0.0')dnl
 58 dnl DAEMON_OPTIONS(`Family=inet6, Name=MSP-v6, Port=submission, M=Ea, Addr=::1')dnl
 59 DAEMON_OPTIONS(`Family=inet,  Name=MSP-v4, Port=submission, M=Ea, Addr=0.0.0.0')dnl
```

변경한 설정을 적용해 `sendmail.cf` 파일 재생성한다.
```bash
> sudo m4 /etc/mail/sendmail.mc > ./sendmail.cf
```

`sendmail.cf` 위치를 이동한다.

```bash
> sudo mv ./sendmail.cf /etc/mail
```

`root` 계정이 아닌 일반 계정으로 메일을 보낼 경우 에러가 발생할 수 있다. 다음과 같이 권한 설정을 한다.

```bash
> sudo chown root:smmsp /etc/mail/sendmail.cf
```

호스트 이름을 수정한다.

`/etc/hosts`
```vim
 10 127.0.1.1       da-dogk domain.com      da-dogk
```

서비스를 시작한다.

```bash
> sudo systemctl start sendmail
```

## Spring Boot

### gradle 추가

`build.gradle`
```gradle
dependencies {
    ...
	implementation('org.springframework.boot:spring-boot-starter-mail:3.1.2')
```

### 설정 추가
`src/main/resources/application.yml`
```yml
spring:
  mail:
    host: localhost
    port: 25
    properties:
      mail:
        debug: false
        smtp:
          starttls:
            enable: false
          auth: false
```

### 코드 추가

`예시 코드`
```java
@Service
@RequiredArgsConstructor
@Log4j2
public class SchoolService {
    private final JavaMailSender javaMailSender;
    ...

    public void sendAuthCodeMail(AuthMailRequest dto) {
        ...
        try {
            ...

            MimeMessage mimeMessage = javaMailSender.createMimeMessage();
            MimeMessageHelper mimeMessageHelper = new MimeMessageHelper(mimeMessage, false, "UTF-8");
            mimeMessageHelper.setTo(dto.getEmail());
            mimeMessageHelper.setFrom("service@dadogk2.duckdns.org");
            mimeMessageHelper.setSubject("다독: 대학생 이메일 인증 코드입니다.");
            mimeMessageHelper.setText("인증 코드: " + code + "\n본 메일 주소는 발송용입니다.");
            javaMailSender.send(mimeMessage);
        } catch (MessagingException e) {
            // TODO: 전송 실패에 대한 예외 처리
        }
    }
```

[전체 예시 코드](https://github.com/Potato-Y/da_dogk/blob/0c405fa4c400a8a20dbc66418b05add95e635986/backend/build.gradle)

---
## 참고 자료

[[Linux/ubuntu] smtp를 이용해 메일보내기](https://mandoo12.tistory.com/13)

[Ubuntu SMTP 서버 sendmail 설치 및 설정하기](https://blog.naver.com/bethbang1004/221845635884)