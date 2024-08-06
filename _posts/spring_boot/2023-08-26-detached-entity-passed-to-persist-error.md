---
title: "[JPA] detached entity passed to persist 오류"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - JPA
    - SpringBoot
---


spring bot test 코드를 작성하고, 실행하는 중에 문제가 발생했다.

`
org.springframework.dao.InvalidDataAccessApiUsageException: detached entity passed to persist
`

기존에 Postman으로 테스트를 할 때에는 문제가 없었는데, test 코드를 작성하여 실행하니 문제가 발생했다.

서핑을 해보니 아래의 블로그 게시글이 보였다.

https://abbo.tistory.com/276
https://bottom-to-top.tistory.com/24

결론적으로 `cascade`의 문제였다. `cascade`를 다시 살펴보면 아래와 같다.

### Cascade Type
- CascadeType.ALL
  - 모든 Cascade를 적용한다.
- CascadeType.RESIST
  - 엔티티를 생성하고, 연관 엔티티를 추가했을 때 `persist()`를 수행하면 연관 엔티티도 함께 `persist()`가 수행된다.
  만약 연관 엔티티가 DB에 등록된 키값을 가지고 있다면 `deatched entity passed to persist Exception`이 발생한다.
- CascadeType.MERGE
  - 트랜잭션이 종료되고 `detach` 상태에서 연관 엔티티를 추가하거나 변경된 이후에 부모 엔티티가 `merge()`를 수행하게 되면 변경사항이 적용된다. 
  (연관 엔티티의 추가 및 수정 모두 적용된다.)
- CascadeType.REMOVE
  - 삭제 시 연관된 엔티티도 같이 삭제된다.
- CascadeType.DETACH
  - 부모 엔티티가 `detach()`를 수행하게 되면, 연관된 엔티티도 `detach()` 상태가 되어 변경사항이 반영되지 않는다.
  
우선 본인의 경우 완전히 잘 못 사용했다.

### 필드 구성
![image](https://github.com/user-attachments/assets/35efa862-81ed-4356-8a3a-60a2560360ee)

그룹별로 게시글을 작성할 수 있고, 그룹이 삭제될 때 게시글이 함께 삭제되고, 첨부파일도 함께 삭제시키고자 했다. 다만 Cascade를 잘못된 필드에서 사용했다. 

`Post.java`의 코드 일부분이다.
```java
    @ManyToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "group_id", nullable = false, updatable = false)
    private Group group; // post를 올릴 group
```
`cascade`가 Group이 아닌 Post에 적용되어 있다. 이로 인해 게시글이 삭제될 때 그룹이 삭제되는 기이한 코드가 되어버렸다. 

### 코드 수정
기존의 Group과 Post Entity를 수정한다. 

[GitHub Group.java 변경점 보기](https://github.com/Potato-Y/Syiary/commit/9f5136f0e25c8b3296a4c9fd1df1b67bdada7e70?diff=split#diff-46055b39a0a9473d05d3f538b3a7bdbe8ad0da09c8f984972dcdfe1409666763)
```java
@OneToMany(mappedBy = "group", cascade = CascadeType.REMOVE)
    private List<Post> posts = new ArrayList<>();
```
위의 내용을 추가했다. 이를 통해 1:N을 이루고, Group이 삭제될 때 Post도 함께 사라지도록 `Cascade`도 설정했다.

[GitHub Post.java 변경점 보기](https://github.com/Potato-Y/Syiary/commit/9f5136f0e25c8b3296a4c9fd1df1b67bdada7e70?diff=split#diff-115b8f5557e46dcab6705b8a42c83345fec31b1bd23af7a3b4798389a059e064)
기존에는 `cascade = CascadeType.ALL`이었는데, 의도한 목적에 맞게 `cascade = CascadeType.REMOVE`으로 수정했다.


### 문제 해결 완료
해당 작업을 통해 의도와 다르게 작동하는 코드를 수정하고, 기존에 누락된 기능도 추가하였다. ~~(그룹을 삭제할 때 포스트도 함께 삭제하는 기능이 누락되어 있었다...)~~

