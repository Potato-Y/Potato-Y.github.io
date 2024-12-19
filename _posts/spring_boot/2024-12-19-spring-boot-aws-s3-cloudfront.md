---
title: "Spring boot AWS S3 + CloudFront 서명된 URL 사용하기 1"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - AWS
    - S3
    - CloudFront
---

# Spring boot AWS S3 + CloudFront 서명된 URL 사용하기

기존 프로젝트에서는 Local 저장소에 파일을 저장했지만, 일단 이는 스케일 아웃을 하기 힘들어진다. 그 외에도 AWS S3의 이점도 많기 때문에 이번에는 Local 저장소가 아닌 S3 버킷에 저장하고자 한다.

그리고, S3 버킷에 저장된 이미지는 CloudFront를 통해 접근할 수 있도록 하려고 한다. 이는 S3에 직접 파일을 요청하는 것보다 빠르면서(엣지 로케이션 캐싱) 비용을 낮출 수 있는 좋은 방법 중 하나이다. 그리고, 서명된 URL을 사용하도록 하려고 한다. 공개된 파일이라면 CloudFront 도메인과 S3 위치를 통해 URL을 쉽게 만들 수 있지만, 특정 일부에게만 파일에 접근이 가능하게 하려면 `Signed URL` 혹은 `Signed Cookie`를 사용해야 한다. 이번 글에서는 특정 사람들에게만 파일에 접근할 수 있도록 하고자 서명된 URL을 사용하고자 한다. 이를 통해 제한된 시간 동안 해당 링크가 있는 사용자만 요청할 수 있게 된다.

## AWS 준비

### S3 버킷 생성

AWS의 S3 페이지로 이동한다. 그리고 `버킷 만들기` 버튼을 통해 새로운 버킷을 만드는 페이지로 이동한다.

![1](https://github.com/user-attachments/assets/27dc7635-b564-473e-8539-e15e264f3106)

그리고 원하는 버킷 이름을 입력한다. 아래의 사진처럼 버킷의 퍼블릭 액서스를 차단하도록 설정한다. 이는 기본값으로 되어있을 것인데, 이번 글에서는 CloudFront를 통해 파일에 접근하도록 할 것이기 때문에 퍼블릭 엑서스는 차단하는 것이다.

![2](https://github.com/user-attachments/assets/a7dc008c-c945-46cf-a999-f96215166fe1)

설정을 완료했다면 버킷을 생성한다. 버킷을 생성하고 들어가면 마치 Cloud 서비스처럼 폴더를 만들고, 파일을 업로드 할 수 있는 모습을 볼 수 있다. 

>참고로 S3는 사실상 폴더의 개념이 없다. 폴더를 흉내 낼 뿐, 실제 폴더가 아니다. `/`를 통해 폴더를 흉내내기 때문에 파일명에 `/`가 포함될 수 없다. 만약 `/`가 포함된 파일명이라면 `:`으로 변경하여 저장되는 모습을 확인할 수 있다.

### CloudFront 설정

이제는 S3 버킷에 저장된 객체에 접근할 수 있도록 해줄 CloudFront를 설정한다. AWS CloudFront 페이지로 이동한다. `배포 생성` 버튼을 클릭하여 배포 생성 페이지로 이동한다.

`Origin domain`은 위에서 생성한 S3를 선택해 주면 된다. 

그리고, 원본 액세스 설정은 `Legacy access identities`으로 한다. 하단의 `원본 액세스 ID`는 `새 OAI 생성` 버튼을 눌러 새로운 `Origin Access Identity`을 생성하고, 버킷 정책을 자동으로 업데이트하도록 `예, 버킷 정책 업데이트`를 선택한다.

권장 옵션과 레거시 옵션 둘 중 무엇을 선택하든 사용자 입장에서는 큰 차이가 없다고 전해 들었다. 때문에 본인의 편의에 맞추어 원하는 옵션을 선택하여 진행하도록 한다.

그리고 하단으로 이동하여 뷰어 액세스 제한 옵션을 `Yes`를 선택한다. `신뢰할 수 있는 인증 유형`은 변경하지 않는다.

### RSA key pair 생성 및 등록

이제 `RSA key pair`를 생성할 것이다. 여기서 생성된 공개 키는 AWS CloudFront의 Key Group에 등록된다. 개인 키는 애플리케이션에서 Signed URL를 생성하는 데 사용된다. 해당 URL에는 `만료 시간`, `IP 주소 제한`, `시작 시간` 등과 같은 정보가 포함될 수 있다. 만들어진 URL에 접속하면 CloudFront에서 공개 키를 사용해 Signed URL의 서명이 유효한지 확인하고, URL에 포함된 정책 조건을 검토하여 요청이 허용 가능한지 확인한다.

1. OpenSSL을 사용하여 길이가 2048비트인 RSA 키 페어를 생성한다. 이름은 본인이 원하는 이름으로 한다.

```bash
openssl genrsa -out aws_ex_private_key.pem 2048
```

2. 위에서 생성한 파일에서 공개 키를 추출한다.

```bash
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

이제 추출한 공개키를 CloudFront에 등록해야 한다. 새 탭을 열어 `CloudFront > Public Key > Create public key` 페이지로 이동한다. 이름을 저장하고 위에서 생성한 `public_key.pem`의 내용을 `Key` 칸에 입력한다. 그리고 `퍼블릭 키 생성` 버튼을 클릭한다.

![3](https://github.com/user-attachments/assets/b88b732e-786c-4d74-a22a-0c9db420520e)

그리고 `CloudFront > Key groups > Create key group`으로 이동하여 아까 생성한 키를 통해 키 그룹을 새로 생성한다.

![4](https://github.com/user-attachments/assets/23049eee-c1fc-4974-bad5-c049c644ee13)

### 키 그룹 선택

다시 배포 생성 페이지 탭으로 이동한다. `키 그룹 선택` 칸 오른쪽의 새로 고침 버튼을 클릭한다. 그러면 위에서 등록한 키 그룹 이름이 보일 것이다. 해당 이름을 선택해 준다.

이후 웹 애플리케이션 방화벽은 비활성화한다. 활성화 시 이용 요금이 청구되는 것으로 알고 있다.

최종 내용은 아래의 사진과 같다.

![5](https://github.com/user-attachments/assets/8d351bdb-c9b0-475a-b15d-9a7a7f276a20)

이제 배포 생성 버튼을 눌러 완료한다.

### IAM 설정

이제 Spring Boot에서 S3와 CloudFront에 접근하기 위한 계정을 생성해주어야 한다. Root 계정을 통해 제어하면 모든 권한을 모두 갖고 있는 것이기 때문에 여러모로 안전하지 못하다. 때문에 S3와 CloudFront에만 접근할 수 있는 계정을 추가로 만들어준다. 

`Identity and Access Management(IAM)` 페이지로 이동한다. 그리고 새로운 `사용자 생성` 버튼을 누른다.

사용자 세부 정보 지정 페이지에서 사용자 이름을 입력한다.

![6](https://github.com/user-attachments/assets/2946b4d5-c482-4d59-ae67-93460daf7dc7)

그리고 권한 설정 페이지에서는 `직접 정책 연결` 항목을 선택하고, 하단의 권한 정책에서 `AmazonS3FullAccess`을 검색한다. 해당 항목을 선택한다. 이름과 같이 S3와 관련된 모든 권한을 갖는데, 빠르게 진행하기 위해 해당 권한을 선택하는 것이다. 만약 실제 서비스에 적용하며, 객체를 저장하고 삭제하는 권한만 갖도록 하려면 `정책 생성`을 통해 생성과 삭제 권한만 가지는 정책을 만들어야 한다.

![7](https://github.com/user-attachments/assets/2dddced6-2c64-47a7-b23a-4eea1d12b3ee)

사진과 같이 설정한 다음에 다음으로 넘어가서 사용자를 생성하도록 한다.

이제 Spring Boot에서 해당 계정을 사용할 수 있도록 엑세스 키를 발급받아야 한다. 여기서 발급받는 키는 외부에 유출되지 않도록 주의해야 하며, Spring Boot의 `application.yml` 파일에 추가해 줄 것이다.

`IAM > 사용자 > [본인이 만든 사용자 이름]`으로 이동하고, `보안 자격 증명` 탭으로 이동한다. 하단에 스크롤 하여 액세스 키 항목의 `액세스 키 만들기` 버튼을 눌러준다.

![8](https://github.com/user-attachments/assets/41dc19aa-a489-4836-aec9-44fd39656056)

그런 다음, `액세스 키 모범 사례 및 대안` 단계에서 `기타` 항목을 선택하고 다음 페이지로 이동한다.

![9](https://github.com/user-attachments/assets/9579d4ca-6a56-4b00-bd94-fcddba1a8815)

`설명 태그 설정 - 선택 사항`에서는 필요한 태그 값을 입력한 다음 `액세스 키 만들기` 버튼을 눌러 키를 생성한다.

다음 페이지로 이동하면 `액세스 키 검색` 단계로 오면 `액세스 키`와 `비밀 액세스 키`를 확인할 수 있다. 해당 키는 해당 페이지를 나가면 다시는 확인할 수 없다. 때문에 안전한 곳에 복사하거나 `.csv` 파일을 다운받아 보관하도록 한다. 그리고 `완료` 버튼을 눌러 완료한다.


## Spring Boot 프로젝트 설정

먼저 Spring Boot 프로젝트를 생성한다. [start.spring.io](https://start.spring.io/) 또는 Idea에서 Spring Boot 프로젝트를 추가하면 된다.

### build.gradle

의존성은 아래의 전체 파일 내용을 참고한다. `com.amazonaws:aws-java-sdk`를 추가해야 AWS의 S3와 CloudFront를 사용할 수 있다.

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.4.0'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'org.example'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.amazonaws:aws-java-sdk:1.12.779'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

[github](https://github.com/Potato-Y/springboot-aws-s3-cloudfront-ex/blob/master/build.gradle)

### application.yml

이제 앞에서 작업한 내용들을 환경 변수에 추가할 차례이다. 다음과 같이 내용을 추가한다.

```yml
spring:
  application:
    name: aws-s3-cloudfront

  datasource:
    url: jdbc:h2:mem:testdb
    username: sa

  h2:
    console:
      enabled: true
      path: /h2-console
      settings:
        web-allow-others: true

aws:
  access-key: ACCESS_KEY
  secret-key: SECRET_KEY
  s3:
    bucket-name: spring-ex
  cloudfront:
    domain: domain.cloudfront.net
    private-key-path: aws/cloudfront/aws_ex_private_key.pem
    key-id: PUBLIC_KEY_ID
```

- `access-key`와 `secret-key` 자리에는 앞서 위에서 얻은 액세스 키를 넣어주면 된다. 
- s3의 `bucket-name`은 앞서 s3 버킷을 생성할 때 사용한 이름을 넣어주면 된다.
- `cloudfront`의 `domain`은 `배포 도메인 이름`을 복사하여 붙여 넣으면 된다. (`CloudFront > 배포 > 해당 배포 항목`으로 이동하면 확인할 수 있다.)
- `private-key-path`는 앞서 생성한 `RSA key pair`에서 추출한 private key의 위치를 넣어주면 된다. 필자는 편의를 위해 `resources.aws.cloudfront` 위치에 넣어두었다. 하지만 커밋 시 함께 공개되지 않도록 주의하도록 한다. (필자는 submodule을 통해 분리하여 관리했다.)
- `key-id`는 `CloudFront > Public key`로 이동하면 확인할 수 있는 `ID` 값이다.


### S3Config

```java
package org.example.awss3cloudfront.config;

import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.regions.Regions;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class S3Config {

    @Value("${aws.access-key}")
    private String accessKey;

    @Value("${aws.secret-key}")
    private String secretKey;

    @Bean
    public AmazonS3 amazonS3() {
        return AmazonS3ClientBuilder.standard()
                .withCredentials(
                        new AWSStaticCredentialsProvider(new BasicAWSCredentials(accessKey, secretKey))
                )
                .withRegion(Regions.AP_NORTHEAST_2)
                .build();
    }
}
```

위 코드를 통해 S3 Client를 생성하고, 해당 객체를 Bean으로 등록한다.

### S3Service

```java
package org.example.awss3cloudfront.aws;

import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.model.ObjectMetadata;
import com.amazonaws.services.s3.model.PutObjectRequest;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class S3Service {

    private final AmazonS3 s3Client;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    public String uploadPostFile(MultipartFile file, String path) throws IOException {
        String fileName = this.getFileName(file, path);
        ObjectMetadata objectMetadata = this.getObjectMetadata(file);

        PutObjectRequest request = new PutObjectRequest(
                bucketName, fileName, file.getInputStream(), objectMetadata
        );

        s3Client.putObject(request);
        return fileName;
    }

    private String getFileName(MultipartFile file, String path) {
        String originalFilename = file.getOriginalFilename();
        String extension = originalFilename.substring(originalFilename.lastIndexOf("."));

        return path + "/" + UUID.randomUUID() + extension;
    }

    private ObjectMetadata getObjectMetadata(MultipartFile file) {
        ObjectMetadata objectMetadata = new ObjectMetadata();

        objectMetadata.setContentLength(file.getSize());
        objectMetadata.setContentType(file.getContentType());

        return objectMetadata;
    }
}
```

전체적으로 코드를 간략하게 살펴보면, 고유한 파일의 이름을 생성하여 S3 버킷에 저장하는 내용이다. `getFileName` 메서드는 path 형식이지만, 실제 S3는 디렉터리 구조를 흉내낸 것이지 실제 폴더가 있는 것이 아니기 때문에 `/path/name/image.png` 형식이어도 결국 객체의 이름인 셈이다.

### CloudFrontService

```java
package org.example.awss3cloudfront.aws;

import com.amazonaws.services.cloudfront.CloudFrontUrlSigner;
import com.amazonaws.services.cloudfront.util.SignerUtils;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Service;

import java.io.File;
import java.io.IOException;
import java.security.spec.InvalidKeySpecException;
import java.util.Date;

@Service
@RequiredArgsConstructor
public class CloudFrontService {

    @Value("${aws.cloudfront.domain}")
    private String domain;

    @Value("${aws.cloudfront.private-key-path}")
    private String privateKeyPath;

    @Value("${aws.cloudfront.key-id}")
    private String keyId;

    public String generateSignedUrl(String objectKey) throws InvalidKeySpecException, IOException {
        Date expiration = new Date(System.currentTimeMillis() + (1000 * 60 * 20));
        File privateKey = new ClassPathResource(privateKeyPath).getFile();

        return CloudFrontUrlSigner.getSignedURLWithCannedPolicy(
                SignerUtils.Protocol.https,
                domain,
                privateKey,
                objectKey,
                keyId,
                expiration
        );
    }
}
```

위 코드는 앞서 저장한 객체를 20분 동안 접근할 수 있는 URL을 만드는 코드이다.

### Post

이제 S3 사용 예시로 게시글을 들어보고자 한다. 게시글에는 텍스트와 이미지를 저장할 수 있다는 점이 예시로 딱 좋은 것 같다.

우선 포스트를 생성할 때 생성 시간을 자동으로 입력되도록 하기 위해 Spring 진입점에 다음과 같이 애너테이션을 추가한다.

```java
@EnableJpaAuditing
@SpringBootApplication
public class AwsS3CloudfrontApplication {

    public static void main(String[] args) {
        SpringApplication.run(AwsS3CloudfrontApplication.class, args);
    }

}
```

`Post.java`
```java
package org.example.awss3cloudfront.post.domain;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Getter
@Entity
@Table(name = "posts")
@NoArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "title", length = 20)
    private String title;

    @Column(name = "content")
    private String content;

    @CreatedDate
    @NotNull
    @Column(name = "create_at", updatable = false)
    private LocalDateTime createAt;

    @LastModifiedDate
    private LocalDateTime updateAt;

    @OneToMany(mappedBy = "post", fetch = FetchType.LAZY)
    private List<PostFile> postFiles = new ArrayList<>();

    @Builder
    public Post(String title, String content) {
        this.title = title;
        this.content = content;
    }

    public Post updatePostFiles(List<PostFile> postFiles) {
        this.postFiles = postFiles;

        return this;
    }
}
```

`PostRepository`
```java
package org.example.awss3cloudfront.post.domain;

import org.springframework.data.jpa.repository.JpaRepository;

public interface PostRepository extends JpaRepository<Post, Long> {
}
```

`PostFile`
```java
package org.example.awss3cloudfront.post.domain;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.OnDelete;
import org.hibernate.annotations.OnDeleteAction;

@Getter
@Entity
@Table(name = "post_files")
@NoArgsConstructor
public class PostFile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "post_id")
    @OnDelete(action = OnDeleteAction.CASCADE)
    private Post post;

    @NotNull
    @Column(name = "file_path", unique = true, updatable = false)
    private String filePath;

    @Builder
    public PostFile(Post post, String filePath) {
        this.post = post;
        this.filePath = filePath;
    }
}
```

`PostFileRepository`
```java
package org.example.awss3cloudfront.post.domain;

import org.springframework.data.jpa.repository.JpaRepository;

public interface PostFileRepository extends JpaRepository<PostFile, Long> {
}
```


#### PostService

`PostService.java`
```java
package org.example.awss3cloudfront.post;

import lombok.RequiredArgsConstructor;
import org.example.awss3cloudfront.aws.S3Service;
import org.example.awss3cloudfront.post.domain.Post;
import org.example.awss3cloudfront.post.domain.PostFile;
import org.example.awss3cloudfront.post.domain.PostFileRepository;
import org.example.awss3cloudfront.post.domain.PostRepository;
import org.example.awss3cloudfront.post.dto.CreatePostRequest;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.util.ArrayList;

@Service
@RequiredArgsConstructor
public class PostService {

    private final S3Service s3Service;
    private final PostRepository postRepository;
    private final PostFileRepository postFileRepository;

    @Transactional
    public Post save(CreatePostRequest request) {
        Post post = postRepository.save(Post.builder()
                .title(request.title())
                .content(request.content())
                .build());

        if (request.isFiles()) {
            ArrayList<PostFile> postFiles = new ArrayList<>();

            for (MultipartFile file : request.files()) {
                String path = post.getId().toString(); // {post_id}/image1.png 와 같은 형식이 되도록

                try {
                    String savePath = s3Service.uploadPostFile(file, path);

                    postFiles.add(postFileRepository.save(PostFile.builder()
                            .post(post)
                            .filePath(savePath)
                            .build()));
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
                post.updatePostFiles(postFiles);
            }
        }

        return post;
    }
}
```

위 코드는 사용자가 요청한 내용을 통해 Post를 저장하고, File이 있는 경우 S3에 저장하고 객체 이름을 filePath로 저장하는 코드이다. 여기에 필요한 다른 클래스들의 내용은 아래에 있다.

`CreatePostRequest.java`
```java
package org.example.awss3cloudfront.post.dto;

import jakarta.validation.constraints.NotNull;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

public record CreatePostRequest(
        @NotNull String title,
        @NotNull String content,
        List<MultipartFile> files
) {

    public boolean isFiles() {
        return files != null && !files.isEmpty();
    }
}
```

#### PostApiController

Post를 저장할 수 있는 Endpoint를 만들어준다.

`PostApiController.java`
```java
package org.example.awss3cloudfront.post;

import lombok.RequiredArgsConstructor;
import org.example.awss3cloudfront.aws.CloudFrontService;
import org.example.awss3cloudfront.post.domain.Post;
import org.example.awss3cloudfront.post.dto.CreatePostRequest;
import org.example.awss3cloudfront.post.dto.PostResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;
import java.security.spec.InvalidKeySpecException;
import java.util.ArrayList;

@RequiredArgsConstructor
@RestController
@RequestMapping("/post")
public class PostApiController {

    private final PostService postService;
    private final CloudFrontService cloudFrontService;

    @PostMapping("")
    public ResponseEntity<PostResponse> createPost(@Validated @ModelAttribute CreatePostRequest request) {
        Post post = postService.save(request);

        ArrayList<String> urls = new ArrayList<>();
        post.getPostFiles().forEach(file -> {
            try {
                cloudFrontService.generateSignedUrl(file.getFilePath());
            } catch (InvalidKeySpecException | IOException e) {
                throw new RuntimeException(e);
            }
        });

        return ResponseEntity.status(HttpStatus.CREATED).body(PostResponse.from(post, urls));
    }
}
```

`PostResponse.java`
```java
package org.example.awss3cloudfront.post.dto;

import org.example.awss3cloudfront.post.domain.Post;

import java.time.LocalDateTime;
import java.util.List;

public record PostResponse(
        Long postId,
        String title,
        String content,
        LocalDateTime createAt,
        LocalDateTime updateAt,
        List<String> files
) {

    public static PostResponse from(Post post, List<String> files) {
        return new PostResponse(
                post.getId(),
                post.getTitle(),
                post.getContent(),
                post.getCreateAt(),
                post.getUpdateAt(),
                files
        );
    }
}
```

#### Postman을 통해 확인하기

이제 파일이 정상적으로 S3 버킷에 저장되고, DB에 해당 값이 저장되는지 확인해 본다.

먼저 Postman을 통해 앞서 만든 URL에 파일과 함께 요청을 날려본다.

![10](https://github.com/user-attachments/assets/cf6fae02-f12a-4924-8883-e52b5106657f)

그러면 위 사진과 같이 Post의 내용과 files의 URL이 표시된다. 해당 URL은 20분 동안 접근할 수 있다. 그리고 해당 링크를 들어가 보면 본인이 선택한 이미지가 표시될 것이다.

이제 DB에도 정상적으로 저장이 되었는지 `localhost:8080/h2-console`로 접속하여 확인해 본다.

![11](https://github.com/user-attachments/assets/cb12f2a4-65f8-4bf9-bbf0-abaac2c2f995)

![12](https://github.com/user-attachments/assets/ba66396b-50ca-4c71-b867-a8d22497ead4)

DB에도 정상적으로 저장된 것을 확인할 수 있다. file path는 S3 버킷에 저장된 위치이며, 고유한 이름을 갖도록 UUID로 파일명이 변경된 것을 확인할 수 있다.

#### Post 조회 API 만들기

`PostService.java`에 다음의 메서드를 추가한다.
```java
    @Transactional(readOnly = true)
    public Post findById(Long id) {
        return postRepository.findById(id).orElseThrow(RuntimeException::new);
    }
```

`PostApiController.java`를 아래와 같이 수정한다. 중복되는 메서드를 분리하고, 조회 Endpoint가 추가되었다.

```java
package org.example.awss3cloudfront.post;

import lombok.RequiredArgsConstructor;
import org.example.awss3cloudfront.aws.CloudFrontService;
import org.example.awss3cloudfront.post.domain.Post;
import org.example.awss3cloudfront.post.dto.CreatePostRequest;
import org.example.awss3cloudfront.post.dto.PostResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;
import java.security.spec.InvalidKeySpecException;
import java.util.ArrayList;
import java.util.List;

@RequiredArgsConstructor
@RestController
@RequestMapping("/post")
public class PostApiController {

    private final PostService postService;
    private final CloudFrontService cloudFrontService;

    @PostMapping("")
    public ResponseEntity<PostResponse> createPost(@Validated @ModelAttribute CreatePostRequest request) {
        Post post = postService.save(request);

        List<String> urls = getFileUrls(post);

        return ResponseEntity.status(HttpStatus.CREATED).body(PostResponse.from(post, urls));
    }

    @GetMapping("/{postId}")
    public ResponseEntity<PostResponse> getPost(@PathVariable Long postId) {
        Post post = postService.findById(postId);
        List<String> urls = getFileUrls(post);

        return ResponseEntity.status(HttpStatus.OK).body(PostResponse.from(post, urls));
    }

    private List<String> getFileUrls(Post post) {
        List<String> urls = new ArrayList<>();
        post.getPostFiles().forEach(file -> {
            try {
                urls.add(cloudFrontService.generateSignedUrl(file.getFilePath()));
            } catch (InvalidKeySpecException | IOException e) {
                throw new RuntimeException(e);
            }
        });

        return urls;
    }
}
```

![13](https://github.com/user-attachments/assets/312a2542-f815-4c10-8792-e68b4a42be34)

위 사진과 같이 GET 요청을 하면 해당 Post의 내용과 20분간 접속할 수 있는 URL이 있는 것을 확인할 수 있다.

---

이번 게시글에서는 S3 버킷에 파일을 업로드를 하고, 조회하는 방법을 확인했다. 다음에는 게시글이 삭제될 때 S3 객체도 함께 삭제하는 내용을 작성해 보고자 한다.

## 참고

[Spring Boot와 CDN을 연결해보자! (1) — cloudFront를 활용한 이미지 캐싱](https://medium.com/@hee98.09.14/spring-boot%EC%99%80-cdn%EC%9D%84-%EC%97%B0%EA%B2%B0%ED%95%B4%EB%B3%B4%EC%9E%90-1-cloudfront%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EC%9D%B4%EB%AF%B8%EC%A7%80-%EC%BA%90%EC%8B%B1-c29cc0f3cbdd)
[AWS CloudFront에 Signed URLs 적용하기](https://velog.io/@leehaeun0/AWS-CloudFront%EC%97%90-Signed-URLs-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0)
