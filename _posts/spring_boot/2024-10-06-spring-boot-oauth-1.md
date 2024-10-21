---
title: "Spring boot OAuth 사용하기 1 - Kakao"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - OAuth 2.0
---

# Spring Boot OAuth 사용하기 - Kakao

요즘에는 대부분의 서비스에서 SNS로 로그인하는 기능을 제공한다. 항상 ID, PW를 통해서만 로그인 기능을 만들어왔지만, 이제는 SNS 로그인을 도전해 봐야 할 때인 것 같다.

> [출처] 본 글은 다음의 내용들을 기반으로 작성되었다. 많은 부분이 흡사하다.
>- https://ddonghyeo.tistory.com/16
>- https://bcp0109.tistory.com/379
>- 감사합니다!!


## 작동 과정 살펴보기

![kakaologin_process_image](https://developers.kakao.com/docs/latest/ko/assets/style/images/kakaologin/kakaologin_process.png)

간단하게 과정을 살펴본다. 자세한 설명은 [참고 블로그](https://ddonghyeo.tistory.com/16) 혹은 [Kakao Developers](https://developers.kakao.com/docs/latest/ko/kakaologin/common#login-seq)를 참고한다.

- 카카오 로그인
  1. 사용자가 카카오 계정을 통한 로그인을 요청한다.
  2. 사용자가 미리 설정된 `client_id`와 `redirect URI`가 포함된 카카오 로그인 페이지에 접속한다.
  3. 사용자가 카카오 로그인에 성공하면 설정된 `redirect URI`로 이동한다. 파라미터에는 `code`가 포함된다.

- 회원 확인 및 등록
  1. 위에서 발급받은 `code`를 통해 사용자의 `access token`과 `refresh token`을 발급받는다.
  2. `access token`을 통해 사용자의 정보를 요청한다.
  3. 받은 정보를 통해 회원가입 또는 로그인을 처리한다.

## 카카오에 애플리케이션 등록 및 설정

### 애플리케이션 추가
https://developers.kakao.com/ 에서 새로운 애플리케이션을 추가한다.

![사진1](https://github.com/user-attachments/assets/a04dd7d6-f658-47a8-b765-e6b63cba47b5)


### 앱 키 확인하기

카카오 로그인 페이지를 연결할 때 사용할 `Key`를 확인한다. `내 애플리케이션 > 앱 설정 > 앱 키`로 이동한 다음 `REST API 키`의 내용을 복사한다. Spring Boot 프로젝트에서 사용한다.

![사진2](https://github.com/user-attachments/assets/a5c02e5c-eda7-4078-ba4f-c8c40ddc3ef4)


### 동의 항목 설정

사용자의 정보를 카카오로부터 제공받기 위해서는 동의 항목을 설정해야 한다. `내 애플리케이션 > 제품 설정 > 카카오 로그인 > 동의항목`으로 이동하여 필요한 항목들을 활성화해 준다.

![사진3](https://github.com/user-attachments/assets/ee3a3a86-271d-476f-a000-6a05b44fc7fe)

아쉽게도 대부분의 정보는 심사를 받은 뒤에 가능하다. 최근에는 `Email`조차 제공 받을 수 없지만 `고유 ID`를 받을 수 있으니, 사용자를 구분함에는 문제가 없다.


### 로그인 활성화 및 Redirect URI 추가

카카오 로그인을 사용하기 위해 `내 애플리케이션 > 제품 설정 > 카카오 로그인`으로 이동하고 활성화 설정 옵션을 `ON`으로 변경한다.

그리고 사용자가 로그인에 성공했을 때 `redirect` 될 `URI`를 설정해야 한다. 우선은 로컬 개발 환경으로 `localhost`를 등록할 것이지만, 앱을 개발한다면 앱 개발자가 제공하는 `URI`를 등록하면 될 것이다.

`Redirect URI 등록` 버튼을 눌러 `http://localhost:8080/auth/callback/kakao`를 추가한다.

![사진4](https://github.com/user-attachments/assets/b53645dc-d947-4c47-80fb-ec5ac4724a13)

사진과 같이 설정되면 된다.


## Spring Boot + Kakao OAuth 2.0

이제 Spring Boot 프로젝트를 진행한다.


### 의존성 설정

`build.gradle`
```gradle
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	implementation 'org.springframework.boot:spring-boot-starter-webflux'

	compileOnly 'org.projectlombok:lombok'
	runtimeOnly 'com.h2database:h2'

	annotationProcessor 'org.projectlombok:lombok'

	testImplementation 'org.springframework.boot:spring-boot-starter-test'
	testImplementation 'io.projectreactor:reactor-test'

	testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
```

카카오 로그인을 통해 받은 `code`를 통해 Kakao 서버에 사용자 정보를 얻기 위해서 `Webflux` 라이브러리의 `WebClient`를 사용한다.

필자는 앱을 통해 로그인 페이지로 연결할 것이기 때문에 웹 페이지가 필요하지 않아 `thymeleaf`는 추가하지 않았다.

만약 로그인 페이지를 개발한다면 다음의 [블로그 글](https://ddonghyeo.tistory.com/16#1-1.%20%EB%A1%9C%EA%B7%B8%EC%9D%B8%20%ED%8E%98%EC%9D%B4%EC%A7%80%20%EB%A7%8C%EB%93%A4%EA%B8%B0-1)을 참고한다.


### REST API Key, Redirect URI 등록

`application.yml`에 앞서 얻은 `REST API 키`와 Redirect URI를 등록한다.

```yml
oauth:
  kakao:
    client-id: d1c80920
    url:
      auth: https://kauth.kakao.com
      api: https://kapi.kakao.com
```


### 사용자 정보를 저장할 Entity 정의

본 프로젝트는 `ID`, `PW`를 사용하지 않고 SNS 로그인만 사용한다는 가정에 진행한다. SNS 로그인은 Kakao가 될 수도 Naver가 될 수도, 그 이외의 것이 될 수 있기 때문에 이를 고려하여 진행한다.

#### Login Type 정의

```java
package com.example.oauth.authentication.domain.oauth;

public enum OAuthProvider {
  KAKAO,NAVER
}
```

회원이 어떤 로그인 타입을 사용했는지를 나타내기 위한 Enum 클래스이다.


#### User Entity

```java
package com.example.oauth.user.domain;

@Table(name = "users")
@Getter
@Entity
@NoArgsConstructor
public class User implements UserDetails {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String serviceId; // OAuth 제공 서비스의 사용자 ID

  private String email;

  private String nickname;

  @Enumerated(EnumType.STRING)
  private OAuthProvider oAuthProvider;

  @Builder
  public User(String serviceId, String email, String nickname, OAuthProvider oAuthProvider) {
    this.serviceId = serviceId;
    this.email = email;
    this.nickname = nickname;
    this.oAuthProvider = oAuthProvider;
  }

  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() {
    return List.of(new SimpleGrantedAuthority("user"));
  }

  @Override // 사용자 id 반환
  public String getUsername() {
    return email;
  }

  @Override // 사용자 패스워드 반환
  public String getPassword() {
    return "sd43adkfl2Kkejrasd12!@#q135";
  }

  @Override
  public boolean isAccountNonExpired() {
    // 만료되었는지 확인하는 로직
    return true; // true -> 만료되지 않음
  }

  @Override
  public boolean isAccountNonLocked() {
    // 계정이 잠금되었는지 확인하는 로직
    return true; // true -> 잠금되지 않음
  }

  @Override
  public boolean isCredentialsNonExpired() {
    // 패스워드가 만료되었는지 확인하는 로직
    return true; // true -> 만료되지 않음
  }

  // 계정 사용 가능 여부 반환

  @Override
  public boolean isEnabled() {
    // 계정이 사용 가능한지 확인하는 로직
    return true; // true -> 사용 가능
  }
}
```

회원 정보를 담는 `User` Entity이다. Kakao의 경우에는 Email을 제공받을 수 없는 상태이기 때문에 고유한 값인 `serviceId` 즉, `access token`을 통해 사용자 정보를 조회할 때 얻을 수 있는 카카오 계정의 고유 id를 저장한다.

> UserDetails를 상속받기 때문에 선언하는 메서드가 많다. 뒤에서 Spring Security를 활용하기 위해 위와 같이 했다.


### 외부 API 요청

#### OAuthLoginParams

OAuth 로그인은 Kakao뿐 아니라 다른 곳에도 사용할 수 있고, 큰 틀이 변하지 않는다. 그 때문에 Interface를 선언하여 공통된 로직을 하나로 처리할 수 있다.

`외부 Access Token 요청` -> `프로필 정보 요청` -> `이메일, 닉네임 가져오기`와 같은 로직들을 말한다.


`OAuthLoginParams`
```java
package com.example.oauth.authentication.domain.oauth;

public interface OAuthLoginParams {

  OAuthProvider oAuthProvider();

  String getCode();
}
```

OAuth 요청을 위한 파라미터 값들을 가진다.


#### KakaoLoginParams
```java
package com.example.oauth.authentication.infra.kakao.dto;

@Getter
@Setter
public class KakaoLoginParams implements OAuthLoginParams {

  private String code;

  @Override
  public OAuthProvider oAuthProvider() {
    return OAuthProvider.KAKAO;
  }
}
```

카카오 API 요청에 필요한 `authorization_code`를 가지고 있는 DTO 클래스다.


#### KakaoTokenResponse

앞서 받는 `Authorization Code`를 통해 타 플랫폼의 `Access Token`을 받아오기 위한 `Response Model`이다.

`KakaoTokenResponse`
```java
package com.example.oauth.authentication.infra.kakao.dto;

@Getter
@NoArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
public class KakaoTokenResponse {

  @JsonProperty("token_type")
  public String tokenType;

  @JsonProperty("access_token")
  public String accessToken;

  @JsonProperty("id_token")
  public String idToken;

  @JsonProperty("expires_in")
  public Integer expiresIn;

  @JsonProperty("refresh_token")
  public String refreshToken;

  @JsonProperty("refresh_token_expires_in")
  public Integer refreshTokenExpiresIn;

  @JsonProperty("scope")
  public String scope;
}
```

#### OAuthInfoResponse

```java
package com.example.oauth.authentication.domain.oauth;

public interface OAuthInfoResponse {

  String getId();

  String getEmail();

  String getNickname();

  OAuthProvider getOAuthProvider();
}
```
`Access Token`으로 요청한 외부 API 프로필 응답 값을 개발하는 서비스의 Model로 변화시키기 위한 인터페이스다.

> email은 카카오에 서비스 인증을 완료한 뒤 가능

`KakaoInfoResponse`
```java
package com.example.oauth.authentication.infra.kakao.dto;

@Getter
@NoArgsConstructor
@JsonIgnoreProperties(ignoreUnknown = true)
public class KakaoInfoResponse implements OAuthInfoResponse {

  // 회원 번호
  @JsonProperty("id")
  private Long id;

  // 자동 연결 설정을 비활성화한 경우만 존재
  // true: 연결 상태, false: 연결 대기 상태
  @JsonProperty("has_signed_up")
  private Boolean hasSignedUp;

  // 서비스에 연결 완료된 시각. UTC
  @JsonProperty("connected_at")
  private Date connectedAt;

  //카카오싱크 간편가입을 통해 로그인한 시각. UTC
  @JsonProperty("synched_at")
  private Date synchedAt;

  //사용자 프로퍼티
  @JsonProperty("properties")
  private HashMap<String, String> properties;

  //카카오 계정 정보
  @JsonProperty("kakao_account")
  private KakaoAccount kakaoAccount;

  //uuid 등 추가 정보
  @JsonProperty("for_partner")
  private Partner partner;

  @Override
  public String getId() {
    return String.valueOf(id);
  }

  @Override
  public String getEmail() {
    return kakaoAccount.email;
  }

  @Override
  public String getNickname() {
    return kakaoAccount.profile.nickName;
  }

  @Override
  public OAuthProvider getOAuthProvider() {
    return OAuthProvider.KAKAO;
  }

  @Getter
  @NoArgsConstructor
  @JsonIgnoreProperties(ignoreUnknown = true)
  public class KakaoAccount {

    //프로필 정보 제공 동의 여부
    @JsonProperty("profile_needs_agreement")
    private Boolean isProfileAgree;

    //닉네임 제공 동의 여부
    @JsonProperty("profile_nickname_needs_agreement")
    private Boolean isNickNameAgree;

    //프로필 사진 제공 동의 여부
    @JsonProperty("profile_image_needs_agreement")
    private Boolean isProfileImageAgree;

    //사용자 프로필 정보
    @JsonProperty("profile")
    private Profile profile;

    //이름 제공 동의 여부
    @JsonProperty("name_needs_agreement")
    private Boolean isNameAgree;

    //카카오계정 이름
    @JsonProperty("name")
    private String name;

    //이메일 제공 동의 여부
    @JsonProperty("email_needs_agreement")
    private Boolean isEmailAgree;

    //이메일이 유효 여부
    // true : 유효한 이메일, false : 이메일이 다른 카카오 계정에 사용돼 만료
    @JsonProperty("is_email_valid")
    private Boolean isEmailValid;

    //이메일이 인증 여부
    //true : 인증된 이메일, false : 인증되지 않은 이메일
    @JsonProperty("is_email_verified")
    private Boolean isEmailVerified;

    //카카오계정 대표 이메일
    @JsonProperty("email")
    private String email;

    //연령대 제공 동의 여부
    @JsonProperty("age_range_needs_agreement")
    private Boolean isAgeAgree;

    //연령대
    //참고 https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#req-user-info
    @JsonProperty("age_range")
    private String ageRange;

    //출생 연도 제공 동의 여부
    @JsonProperty("birthyear_needs_agreement")
    private Boolean isBirthYearAgree;

    //출생 연도 (YYYY 형식)
    @JsonProperty("birthyear")
    private String birthYear;

    //생일 제공 동의 여부
    @JsonProperty("birthday_needs_agreement")
    private Boolean isBirthDayAgree;

    //생일 (MMDD 형식)
    @JsonProperty("birthday")
    private String birthDay;

    //생일 타입
    // SOLAR(양력) 혹은 LUNAR(음력)
    @JsonProperty("birthday_type")
    private String birthDayType;

    //성별 제공 동의 여부
    @JsonProperty("gender_needs_agreement")
    private Boolean isGenderAgree;

    //성별
    @JsonProperty("gender")
    private String gender;

    //전화번호 제공 동의 여부
    @JsonProperty("phone_number_needs_agreement")
    private Boolean isPhoneNumberAgree;

    //전화번호
    //국내 번호인 경우 +82 00-0000-0000 형식
    @JsonProperty("phone_number")
    private String phoneNumber;

    //CI 동의 여부
    @JsonProperty("ci_needs_agreement")
    private Boolean isCIAgree;

    //CI, 연계 정보
    @JsonProperty("ci")
    private String ci;

    //CI 발급 시각, UTC
    @JsonProperty("ci_authenticated_at")
    private Date ciCreatedAt;

    @Getter
    @NoArgsConstructor
    @JsonIgnoreProperties(ignoreUnknown = true)
    public class Profile {

      //닉네임
      @JsonProperty("nickname")
      private String nickName;

      //프로필 미리보기 이미지 URL
      @JsonProperty("thumbnail_image_url")
      private String thumbnailImageUrl;

      //프로필 사진 URL
      @JsonProperty("profile_image_url")
      private String profileImageUrl;

      //프로필 사진 URL 기본 프로필인지 여부
      //true : 기본 프로필, false : 사용자 등록
      @JsonProperty("is_default_image")
      private String isDefaultImage;

      //닉네임이 기본 닉네임인지 여부
      //true : 기본 닉네임, false : 사용자 등록
      @JsonProperty("is_default_nickname")
      private Boolean isDefaultNickName;
    }
  }

  @Getter
  @NoArgsConstructor
  @JsonIgnoreProperties(ignoreUnknown = true)
  public static class Partner {

    //고유 ID
    @JsonProperty("uuid")
    private String uuid;
  }
}

```
[출처](https://ddonghyeo.tistory.com/16)의 개발자분께서 정리해 주셨다.

`Model`에 없는 값이 있을 수 있기 때문에 `@JsonIgnoreProperties(ignoreUnknown = true)`를 사용한다.

애너테이션을 사용했기 때문에 필요 없는 값은 지워도 무관하다.


#### OAuthApiService

```java
package com.example.oauth.authentication.domain.oauth;

public interface OAuthApiService {

  OAuthProvider oAuthProvider();

  String requestAccessToken(OAuthLoginParams params);

  OAuthInfoResponse requestOAuthInfo(String accessToken);
}
```

OAuth 요청을 위한 Service 클래스이다. 

- `oAuthProvider`: OAuth 제공 업체
- `requestAccessToken`: `Authorization Code`를 기반으로 인증 API를 요청하여 `Access Token`을 조회
- `requestOAuthInfo`: `Access Token`을 기반으로 프로필 정보를 조회. email, nickname 정보를 가져옴.


#### KakaoApiService

카카오톡 서버로 api를 요청한다. HTTP 요청은 `Webflux` 라이브러리의 `WebClient`를 통해 구현한다.

`KakaoApiService`
```java
package com.example.oauth.authentication.infra.kakao;

@Slf4j
@Service
@RequiredArgsConstructor
public class KakaoApiService implements OAuthApiService {

  private static final String GRANT_TYPE = "authorization_code";

  @Value("${oauth.kakao.url.auth}")
  private String authUrl;

  @Value("${oauth.kakao.url.api}")
  private String apiUrl;

  @Value("${oauth.kakao.client-id}")
  private String clientId;

  @Override
  public OAuthProvider oAuthProvider() {
    return OAuthProvider.KAKAO;
  }

  @Override
  public String requestAccessToken(OAuthLoginParams params) {
    KakaoTokenResponse dto = WebClient.create(authUrl).post()
        .uri(uriBuilder -> uriBuilder
            .scheme("https")
            .path("/oauth/token")
            .queryParam("grant_type", GRANT_TYPE)
            .queryParam("client_id", clientId)
            .queryParam("code", params.getCode())
            .build())
        .header(HttpHeaders.CONTENT_TYPE,
            HttpHeaderValues.APPLICATION_X_WWW_FORM_URLENCODED.toString())
        .retrieve()
        // TODO : Custom Exception
        .onStatus(HttpStatusCode::is4xxClientError,
            clientResponse -> Mono.error(new RuntimeException("Invalid Parameter")))
        .onStatus(HttpStatusCode::is5xxServerError,
            clientResponse -> Mono.error(new RuntimeException("Internal Server Error")))
        .bodyToMono(KakaoTokenResponse.class)
        .block();

    log.info("Access Token: {}", dto.getAccessToken());
    log.info("Refresh Token: {}", dto.getRefreshToken());
    //제공 조건: OpenID Connect가 활성화 된 앱의 토큰 발급 요청인 경우 또는 scope에 openid를 포함한 추가 항목 동의 받기 요청을 거친 토큰 발급 요청인 경우
    log.info("Id Token: {}", dto.getIdToken());
    log.info("Scope: {}", dto.getScope());

    return dto.getAccessToken();
  }

  @Override
  public OAuthInfoResponse requestOAuthInfo(String accessToken) {
    KakaoInfoResponse userInfo = WebClient.create(apiUrl).get()
        .uri(uriBuilder -> uriBuilder
            .scheme("https")
            .path("/v2/user/me")
            .build(true))
        .header(HttpHeaders.AUTHORIZATION, "Bearer " + accessToken) // access token 인가
        .header(HttpHeaders.CONTENT_TYPE,
            HttpHeaderValues.APPLICATION_X_WWW_FORM_URLENCODED.toString())
        .retrieve()
        // TODO : Custom Exception
        .onStatus(HttpStatusCode::is4xxClientError,
            clientResponse -> Mono.error(new RuntimeException("Invalid Parameter")))
        .onStatus(HttpStatusCode::is5xxServerError,
            clientResponse -> Mono.error(new RuntimeException("Internal Server Error")))
        .bodyToMono(KakaoInfoResponse.class)
        .block();

    log.info("Auth ID: {} ", userInfo.getId());
    log.info("NickName: {} ", userInfo.getKakaoAccount().getProfile().getNickName());
    log.info("ProfileImageUrl: {} ", userInfo.getKakaoAccount().getProfile().getProfileImageUrl());

    return userInfo;
  }
}
```

#### RequestOAuthInfoService

상단에서 만든 `OAuthApiService`를 사용하는 `Service` 클래스다. 예를 들어, `KakaoApiService`, `NaverApiService`를 직접 주입 받아 사용하면 중복 코드가 많아지지만, `List<OAuthApiClient>`를 주입 받아서 `Map`으로 만들어 편리하게 사용한다.

> `List<interface>`를 주입받으면 해당 인터페이스의 구현체들이 전부 List에 담겨서 온다.

`RequestOAuthInfoService`
```java
package com.example.oauth.authentication.domain.oauth;

@Component
public class RequestOAuthInfoService {

  private final Map<OAuthProvider, OAuthApiService> clients;

  public RequestOAuthInfoService(List<OAuthApiService> clients) {
    this.clients = clients.stream().collect(
        Collectors.toUnmodifiableMap(OAuthApiService::oAuthProvider, Function.identity())
    );
  }

  public OAuthInfoResponse request(OAuthLoginParams params) {
    OAuthApiService oAuthApiService = clients.get(params.oAuthProvider());
    String accessToken = oAuthApiService.requestAccessToken(params);
    return oAuthApiService.requestOAuthInfo(accessToken);
  }
}
```

#### 중간 작동 확인해보기

간단하게 테스트 하려면 다음과 같이 `Controller`를 추가한다.

```java
package com.example.oauth.authentication.application;

@RestController
@RequiredArgsConstructor
@RequestMapping("/auth")
public class AuthApiController {

  private final RequestOAuthInfoService requestOAuthInfoService;

  @GetMapping("/callback/kakao")
  public ResponseEntity<?> loginKakao(KakaoLoginParams params) {
    requestOAuthInfoService.request(params);
    return ResponseEntity.ok().build();
  }
}
```

테스트용 URL은 다음과 같다.
```
https://kauth.kakao.com/oauth/authorize?client_id=${CLIENT_ID}&redirect_uri=	
${REDIRECT_URI}/auth/callback/kakao&response_type=code
```

해당 링크로 들어가면 Kakao 로그인 페이지(동의 페이지)가 나오고, 동의가 완료되면 리다이렉트 페이지로 이동된다.

Redirect URI에 `authorization code`가 파라미터로 포함되며, Spring Boot에서 이후의 로직이 수행된다. 로그를 통해 정상적으로 작동했는지 확인하면 된다.

> 리다이렉트 페이지(`localhost:8080/...?code=~`)에서 새로고침을 하면 오류가 발생한다. 해당 로직은 `authorization code` 당 1번만 가능하다.


## Spring Security + 회원 추가

이제 SNS를 통해 로그인한 사용자 정보를 DB에 저장하고 JWT를 발급하여 인증 및 인가를 한다.

해당 항목부터는 Spring Security와 JWT 설명 글이 아니기 때문에 간략하게 코드 중심으로 이루어진다.

JWT 구현에 대한 자세한 내용은 

[spring-boot-jwt-tutorial 저장소](https://github.com/SilverNine/spring-boot-jwt-tutorial)와 Inflearn 강의 `정은구님의 Spring Boot JWT Tutorial`

또는 

신선영님의 `스프링 부트 3 백엔드 개발자 되기 자바편`을 참고하는 것이 좋다.


### Spring Boot Security + JWT

#### gradle.build

파일에 `Spring Boot Security` 의존성을 추가한다.

`gradle.build`
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'io.jsonwebtoken:jjwt:0.9.1' // 자바 jwt 라이브러리
    implementation 'org.springframework.boot:spring-boot-starter-validation' // validation
    implementation 'javax.xml.bind:jaxb-api:2.3.1' // XML 문서와 JAVA 객체 간 매핑을 자동화

    ...
}
```

#### application.yml

H2 관련 설정과 Jwt에 사용할 값들을 설정값에 추가한다.

`application.yml`
```yml
spring:
  devtools:
    livereload:
      enabled: true # 프론트 코드 변경시 자동 적용
    restart:
      enabled: true # 코드 변경시 자동 재시작

  datasource:
    url: jdbc:h2:mem:testdb
    username: sa

  h2:
    console:
      enabled: true
      path: /h2-console
      settings:
        web-allow-others: true

jwt:
  issuer: admin@email.com
  secret_key: sample-key
```

#### JwtProperties

```java
package com.example.oauth.config.jwt;

@Setter
@Getter
@Component
@ConfigurationProperties("jwt") // Java class에 properties 값을 가져와 사용하는 애너테이션
public class JwtProperties {

  private String issuer;
  private String secretKey;
}
```

`application.yml`에서 jwt에 대한 설정값을 가져온다.

#### TokenProvider

```java
package com.example.oauth.config.jwt;

/**
 * 토큰 생성, 토큰 유효성 검사, 토큰에서 정보 추출하는 클래스
 */
@Slf4j
@RequiredArgsConstructor
@Service
public class TokenProvider {

  private final JwtProperties jwtProperties;
  private final UserDetailService userDetailService;

  /**
   * JWT 토큰 생성
   *
   * @param user      user 객체
   * @param expiredAt 유효 시간
   * @return Token
   */
  public String generateToken(User user, Duration expiredAt) {
    Date now = new Date();
    return makeToken(new Date(now.getTime() + expiredAt.toMillis()), user);
  }

  /**
   * JWT 토큰을 만들어 반환
   *
   * @param expiry 유효 기간
   * @param user   유저 객체
   * @return Token
   */
  private String makeToken(Date expiry, User user) {
    Date now = new Date();

    return Jwts.builder().setHeaderParam(Header.TYPE, Header.JWT_TYPE) // 헤더 type: JWT
        // 내용 iss: propertise에서 가져온 값
        .setIssuer(jwtProperties.getIssuer()).setIssuedAt(now) // 내용 isa: 현재 시간
        .setExpiration(expiry) // 내용 exp: expiry 멤버 변수값
        .setSubject(user.getId().toString()) // 내용 sub: User id
        .claim("email", user.getEmail()) // 클래임 id: User email
        .claim("nickname", user.getNickname())
        // 서명: 비밀값과 함께 해시값을 HS256 방식으로 암호화
        .signWith(SignatureAlgorithm.HS256, jwtProperties.getSecretKey()).compact();
  }

  /**
   * 유효한 토큰인지 확인
   *
   * @param token Token
   * @return boolean true: 검증 성공, false: 검증 실패
   */
  public boolean validToken(String token) {
    try {
      Jwts.parser().setSigningKey(jwtProperties.getSecretKey()) // 비밀값으로 복호화
          .parseClaimsJws(token);

      return true;
    } catch (Exception e) { // 복호화 과정에서 오류가 발생할 경우 false 반환
      log.warn("validToken. Token 검증 실패. token={}", token);
      return false;
    }
  }

  /**
   * 토큰 기반으로 인증 정보를 가져오는 메서드
   *
   * @param token Token
   * @return User 인증 정보
   */
  public Authentication getAuthentication(String token) {
    Claims claims = getClaims(token);
    Set<SimpleGrantedAuthority> authorities = Collections.singleton(
        new SimpleGrantedAuthority("ROLE_USER"));

    Long userId = Long.parseLong(claims.getSubject()); // 토큰에서 subject를 userId로 사용
    UserDetails userDetails = userDetailService.loadUserById(userId); // ID로 사용자 조회

    return new UsernamePasswordAuthenticationToken(userDetails, token, authorities);
  }

  /**
   * 토큰 기반으로 유저 ID를 가져오는 메서드
   *
   * @param token Token
   * @return (Long) user_id
   */
  public Long getUserId(String token) {
    Claims claims = getClaims(token);
    return Long.parseLong(claims.getSubject());
  }

  /**
   * 토큰에서 body 부분 추출
   *
   * @param token Token
   * @return Claims
   */
  private Claims getClaims(String token) {
    return Jwts.parser() // 클레임 조회
        .setSigningKey(jwtProperties.getSecretKey()).parseClaimsJws(token).getBody();
  }
}
```
Token을 관리하는 내용을 가지고 있다. 토큰을 생성하고, 검증하고, 토큰 속 정보를 뽑아오는 등 작업을 수행한다.


#### TokenAuthenticationFilter

```java
package com.example.oauth.config;

@RequiredArgsConstructor
public class TokenAuthenticationFilter extends OncePerRequestFilter {

  private final static String HEADER_AUTHORIZATION = "Authorization";
  private final static String TOKEN_PREFIX = "Bearer ";
  private final TokenProvider tokenProvider;

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // 요청 헤더의 Authentication 키의 값 조회
    String authorizationHeader = request.getHeader(HEADER_AUTHORIZATION);
    // 가져온 값에서 접두사 제거
    String token = getAccessToken(authorizationHeader);

    // 가져온 토큰이 유효한지 확인하고, 유효한 때는 인증 정보를 설정
    if (tokenProvider.validToken(token)) {
      Authentication authentication = tokenProvider.getAuthentication(token);
      SecurityContextHolder.getContext().setAuthentication(authentication);
    }

    filterChain.doFilter(request, response);
  }

  /**
   * 정상적인 토큰일 경우 접두사를 제거
   */
  private String getAccessToken(String authorizationHeader) {
    if (authorizationHeader != null && authorizationHeader.startsWith(TOKEN_PREFIX)) {
      return authorizationHeader.substring(TOKEN_PREFIX.length());
    }

    return null;
  }
}
```

들어온 요청의 헤더에서 토큰을 추출하여 검증하는 과정을 거친다. 검증에 실패하면 해당 요청은 이후 작업을 수행할 수 없다.


#### WebSecurityConfig

```java
package com.example.oauth.config;

@EnableWebSecurity
@EnableMethodSecurity
@Configuration
@RequiredArgsConstructor
public class WebSecurityConfig {

  private final TokenProvider tokenProvider;

  @Bean
  public WebSecurityCustomizer configure() {
    // Spring Security 기능 비활성화
    return web -> web.ignoring().requestMatchers("/h2-console/**");
  }

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http,
      RequestOAuthInfoService requestOAuthInfoService) throws Exception {
    return http.csrf(AbstractHttpConfigurer::disable).httpBasic(AbstractHttpConfigurer::disable)
        .formLogin(AbstractHttpConfigurer::disable).logout(AbstractHttpConfigurer::disable)

        // JWT 사용을 위해 세션 사용 비활성화
        .sessionManagement(
            session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

        // 헤더를 확인하는 커스텀 필터 추가
        .addFilterBefore(tokenAuthenticationFilter(), UsernamePasswordAuthenticationFilter.class)

        .authorizeHttpRequests(authz -> authz
            // 로그인, 회원가입, 토큰 갱신을 제외한 api는 인증을 하도록 설정
            .requestMatchers("/auth/callback/**", "/h2-console/**").permitAll()
            .anyRequest().permitAll())

        .build();
  }

  @Bean
  public BCryptPasswordEncoder bCryptPasswordEncoder() {
    return new BCryptPasswordEncoder();
  }

  @Bean
  public TokenAuthenticationFilter tokenAuthenticationFilter() {
    return new TokenAuthenticationFilter(tokenProvider);
  }
}
```

`Spring Security` 설정을 진행한다. 우선 `H2` DB에 데이터가 잘 저장되는지 확인할 수 있도록 `/h2-console/**` 경로에 대해 시큐리티 기능을 비활성화한다.

세션 기능을 사용할 것이 아니기 때문에 그에 대해 설정 하고, Token이 없어도 접근할 수 있어야 하는 경로에 대해 예외 처리를 한다. 그 이외에는 `TokenAuthenticationFilter`를 지나가도록 설정한다.


#### TokenDto

```java
package com.example.oauth.authentication.token.dto;

public class TokenDto {

  @Getter
  @Setter
  public static class TokenRequest {

    @NotBlank
    private String accessToken;
    @NotBlank
    private String refreshToken;
  }

  @AllArgsConstructor
  @NoArgsConstructor
  @Getter
  @Setter
  public static class TokenResponse {

    private String refreshToken;
    private String accessToken;
  }

}
```

Token에 대한 정보를 담는 DTO Class이다.


#### TokenService

```java
package com.example.oauth.authentication.token;

@RequiredArgsConstructor
@Service
public class TokenService {

  private static final Duration REFRESH_TOKEN_DURATION = Duration.ofDays(14);
  private static final Duration ACCESS_TOKEN_DURATION = Duration.ofDays(1);

  private final TokenProvider tokenProvider;
  private final AuthenticationManagerBuilder authenticationManagerBuilder;
  private final UserService userService;

  /**
   * 기존 Refresh Token을 통해 새로운 Access Token과 Refresh Token을 생성
   *
   * @param dto TokenRequest
   * @return token
   */
  public TokenResponse updateTokenSet(TokenRequest dto) {
    // 토큰 유효성 검사에 실패하면 예외 발생
    if (!tokenProvider.validToken(dto.getRefreshToken())) {
      throw new IllegalArgumentException("Unexpected token");
    }

    Long userId = tokenProvider.getUserId(dto.getRefreshToken());
    User user = userService.findById(userId);

    String accessToken = tokenProvider.generateToken(user, ACCESS_TOKEN_DURATION);
    String refreshToken = tokenProvider.generateToken(user, REFRESH_TOKEN_DURATION);

    return new TokenResponse(accessToken, refreshToken);
  }

  /**
   * 새로운 로그인을 통해 Access Token과 Refresh Token을 생성
   *
   * @param userId userId
   * @return AuthenticateResponse
   */
  public TokenResponse createNewTokenSet(Long userId) {
    User user = userService.findById(userId);

    // refresh token 생성
    String refreshToken = tokenProvider.generateToken(user, REFRESH_TOKEN_DURATION);
    // access token 생성
    String accessToken = tokenProvider.generateToken(user, ACCESS_TOKEN_DURATION);

    return new TokenResponse(accessToken, refreshToken);
  }
}
```

새로운 `AccessToken`과 `RefreshToken`을 발급하는 기능을 수행한다. 이 부분은 형식만 갖추어 설정했으니, 각자의 입맛에 맞게 변경해야 한다.


#### TokenApiController

```java
package com.example.oauth.authentication.token;

@RequiredArgsConstructor
@RestController
@RequestMapping("/auth")
public class TokenApiController {

  private final TokenService tokenService;

  @PostMapping("/token") // refresh 토큰을 통해 access 토큰 재발급
  public ResponseEntity<TokenResponse> token(@RequestBody TokenRequest request) {
    TokenResponse tokenResponse = tokenService.updateTokenSet(request);

    return ResponseEntity.status(HttpStatus.OK).body(tokenResponse);
  }
}
```

위에서 선언한 `TokenService`의 Token 재발급에 접근할 수 있는 `Controller Class`이다.


#### UserResponse

```java
package com.example.oauth.user.dto;

@NoArgsConstructor
@AllArgsConstructor
@Getter
public class UserResponse {

  private Long userId;
  private String email;
  private String nickname;

  public UserResponse(User user) {
    this.userId = user.getId();
    this.email = user.getEmail();
    this.nickname = user.getNickname();
  }
}
```

로그인을 완료한 사용자의 정보를 반환할 DTO 객체이다.


#### CurrentUserProvider

```java
package com.example.oauth.authentication;

@RequiredArgsConstructor
@Component
public class CurrentUserProvider {

  private final UserRepository userRepository;

  public User getCurrentUser() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    UserDetails userDetails = (UserDetails) authentication.getPrincipal();
    User user = userRepository.findByEmail(userDetails.getUsername()).get();

    return user;
  }
}
```

Security에 등록된 사용자 정보를 조회하여 반환한다. 자주 사용될 메서드라 생각하여 따로 분리하였다.

#### UserService

```java
package com.example.oauth.user;

@RequiredArgsConstructor
@Service
public class UserService {

  private final UserRepository userRepository;
  private final CurrentUserProvider currentUserProvider;

  public User findById(Long userId) {
    return userRepository.findById(userId)
        .orElseThrow(() -> new IllegalArgumentException("Unexpected user"));
  }

  public User findByEmail(String email) {
    return userRepository.findByEmail(email)
        .orElseThrow(() -> new IllegalArgumentException("Unexpected user"));
  }

  public User createUser(OAuthInfoResponse oAuthInfoResponse) {
    User user = User.builder()
        .serviceId(oAuthInfoResponse.getId())
        .oAuthProvider(oAuthInfoResponse.getOAuthProvider())
        .email(oAuthInfoResponse.getEmail())
        .nickname(oAuthInfoResponse.getNickname())
        .build();

    return userRepository.save(user);
  }

  public UserResponse getMyAccount() {
    return new UserResponse(currentUserProvider.getCurrentUser());
  }
}
```

처음 로그인하는 사용자의 정보를 저장하고, 저장된 사용자의 정보를 가져오는 클래스이다.


#### OAuthLoginService

```java
package com.example.oauth.authentication.application;

@Service
@RequiredArgsConstructor
public class OAuthLoginService {

  private final UserService userService;
  private final TokenService tokenService;
  private final UserRepository userRepository;
  private final RequestOAuthInfoService requestOAuthInfoService;

  public TokenResponse login(OAuthLoginParams params) {
    OAuthInfoResponse oAuthInfoResponse = requestOAuthInfoService.request(params);
    Long userId = findOrCreateUser(oAuthInfoResponse);

    return tokenService.createNewTokenSet(userId);
  }

  private Long findOrCreateUser(OAuthInfoResponse oAuthInfoResponse) {
    return userRepository.findByServiceId(oAuthInfoResponse.getId())
        .map(User::getId)
        .orElseGet(() -> userService.createUser(oAuthInfoResponse).getId());
  }
}
```

OAuth를 통해 로그인 요청이 들어올 때, OAuth 제공사로부터 사용자 정보를 조회하고, 만약 저장된 사용자가 없다면 새로 생성한다.


#### AuthApiController 수정

```java
package com.example.oauth.authentication.application;

@RestController
@RequiredArgsConstructor
@RequestMapping("/auth")
public class AuthApiController {

  private final OAuthLoginService oAuthLoginService;

  @GetMapping("/callback/kakao")
  public ResponseEntity<?> loginKakao(KakaoLoginParams params) {
    return ResponseEntity.status(HttpStatus.OK).body(oAuthLoginService.login(params));
  }
}
```

기능 테스트를 위해 만들었던 `AuthApiController` 클래스를 수정한다. 이제는 카카오 로그인을 할 경우 기존 사용자의 경우 새로운 Token을 제공, 처음 로그인한 사용자의 경우 계정 정보를 저장한다.


#### UserController

```java
package com.example.oauth.user;

@RestController
@RequiredArgsConstructor
@RequestMapping("/v1/users")
public class UserController {

  private final UserService userService;

  @GetMapping("/me")
  public ResponseEntity<?> getMyAccount() {
    return ResponseEntity.status(HttpStatus.OK).body(userService.getMyAccount());
  }
}
```

## 작동 테스트

이제 제대로 정상 작동하는지 확인할 차례이다. 브라우저를 통해 Kakao 로그인 과정을 거치면 `http://localhost:8080/auth/callback/kakao`로 리다이렉트 되며 token이 등장한다.

```json
{
    "refreshToken": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhZG1pbkBlbWFpbC5jb20iLCJpYXQiOjE3Mjk1MjM...",
    "accessToken": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJhZG1pbkBlbWFpbC5jb20iLCJpYXQiOjE3Mjk1MjM..."
}
```
여기서 `access Token`을 통해 `UserController`에 등록된 경로로 요청하면 내 서비스에 저장된 계정 정보를 조회할 수 있다.

Poatman을 통해 token을 추가하여 요청하면 다음과 같은 결과를 확인할 수 있다.

```json
{
    "userId": 1,
    "email": null,
    "nickname": "카카오톡 닉네임"
}
```

지금은 서비스를 인증받은 것이 아니기 때문에 email이 null로 뜬다. 어찌되든 해당 이메일은 OAuth 제공 사이트에 따라 중간에 변경될 수 있기 때문에 회원을 고유하게 구분하는 데 사용하진 않아야 할 것 같다.


## 마무리

이 외에도 더 작업할 내용들이 있다. OAuth API에 사용할 `Token`을 저장하지 않고 있다. 만약 후에 OAuth 제공사로부터 사용자 정보를 가져와야 한다면, `Token`을 저장하고 주기적으로 `Token`을 업데이트해 주어야 할 것이다. 또, 웹 페이지가 아닌 앱에 제공한다면 앱 개발자와 의논해야 할 것이다. 사용자의 편의를 위해 `SDK`를 사용할 수도 있고, 리다이렉트를 앱으로 지정하여 앱과 통신해야 할 수 있다.