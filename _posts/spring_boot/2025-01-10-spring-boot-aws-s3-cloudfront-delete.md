---
title: "Spring boot - AWS S3 객체 삭제하기"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - SpringBoot
    - AWS
    - S3
---

[이전 게시글](https://potato-y.github.io/springboot/spring-boot-aws-s3-cloudfront/)에 이어서 이번에는 S3 객체를 삭제하는 방법을 다룬다. Post 단위로 이미지 파일을 저장했기 때문에 디렉터리 단위로 삭제하고자 한다. (물론 이전 게시글에서 설명했듯이 실제 디렉터리 구조는 아니다.)

### S3Service

S3 Service에 디렉터리 위치의 모든 객체를 삭제하는 코드를 작성한다.

`S3Service`
```java
@Service
@RequiredArgsConstructor
public class S3Service {

    ...

    public void deleteFolder(String folderPath) {
        if (!folderPath.endsWith("/")) {
            folderPath += "/";
        }

        ObjectListing objectListing = s3Client.listObjects(bucketName, folderPath); // 목록 가져오기

        // 모든 객체 삭제
        while (true) {
            for (S3ObjectSummary objectSummary : objectListing.getObjectSummaries()) {
                s3Client.deleteObject(bucketName, objectSummary.getKey());
            }

            // 다음 페이지가 있으면 계속 진행
            if (objectListing.isTruncated()) {
                objectListing = s3Client.listNextBatchOfObjects(objectListing);
            } else {
                break;
            }
        }
    }
}
```

### PostService

이제 특정 post의 id를 통해 해당 디렉터리의 s3 객체를 모두 삭제하고, post를 삭제하는 기능을 추가한다.

```java
@Service
@RequiredArgsConstructor
public class PostService {

    ...

    @Transactional
    public void deletePost(Long postId) {
        Post post = postRepository.findById(postId).orElseThrow(() -> new RuntimeException("Post not found"));

        if (!post.getPostFiles().isEmpty()) {
            String path = post.getId().toString(); // {post_id}/image1.png 와 같은 형식이 되도록
            s3Service.deleteFolder(path);
        }

        postRepository.deleteById(postId);
    }
}
```

### PostApiController

이제 Endpoint를 추가한다.

`PostApiController`
```java
@RequiredArgsConstructor
@RestController
@RequestMapping("/post")
public class PostApiController {

    ...

    @DeleteMapping("/{postId}")
    public ResponseEntity<Void> deletePost(@PathVariable Long postId) {
        postService.deletePost(postId);

        return ResponseEntity.status(HttpStatus.NO_CONTENT).build();
    }
}
```

이제 해당 api를 테스트한 뒤에 AWS S3 버킷을 확인해 보면 관련 객체들이 삭제된 것을 확인할 수 있다.

위 코드가 담겨있는 저장소: https://github.com/Potato-Y/springboot-aws-s3-cloudfront-ex/tree/dc6646f8b57e4dd165a48e83286713b22e826541

<img width="780" alt="image" src="https://github.com/user-attachments/assets/0bbd678b-a358-4bc6-89ec-87e02e0c09dc" />

<img width="678" alt="image" src="https://github.com/user-attachments/assets/abba8998-f551-45db-86a6-437b45809105" />
