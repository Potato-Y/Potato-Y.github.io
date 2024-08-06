---
title: "[JPA] detached entity passed to persist 오류"

toc: true
toc_sticky: true

categories:
    - SpringBoot
tags:
    - Java
    - JUnit
    - SpringBoot
---

분명 다른 클래스에서 작업할 때 Intellij의 자동으로 추가하는 improt 기능이 알아서 해준 것 같은데.. 지금은 오류를 뱉는다.

![image](https://github.com/user-attachments/assets/ae86ca48-f89f-4c3a-9eb1-325ff8025bd9)

`Cannot resolve method 'assertThat(HttpStatus)'`
라고 오류를 표시한다. 지금은 improt가 

```java
import static org.junit.Assert.assertThat;
```
로 되어있는데, 이것 말고 아래를 추가해 준다.

```java
import static org.assertj.core.api.Assertions.assertThat;
```

관련 내용 https://www.goodsource.co.kr/44