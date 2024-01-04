---
title: "Mac 파일 확장 속성 확인하고 제거하기"

toc: true
toc_sticky: true

categories:
    - dev_env
tags:
    - Mac
    - ssh
---

맥북으로 `ssh` 접속을 시도하는 중 계속해서 `key`의 권한 문제가 발생했다.

```bash
> ssh -i key user_name@domain
Load key "ubuntu": invalid format
Load key "/Users/-/Desktop/": invalid format
user_name@domain: Permission denied (publickey).
```

위의 내용만으로는 구글을 통해 원인을 찾기 힘들었다. 무엇이 문제인지 확인하기 위해 `ls -al` 명령어를 통해 파일의 상태를 확인해봤다.

```bash
> ls -al
total 40
drwx------@ 10 -  staff    320 12  3 00:23 .
drwx------@ 36 -  staff   1152 12  9 14:09 ..
-rw-------@  1 -  staff   1675  8 22  2022 ssh-key-2022-08-22.key
-rw-------@  1 -  staff    399  8 22  2022 ssh-key-2022-08-22.key.pub
```
위와 같이 `600`이지만 key 파일에서 권한 문제가 발생했다. 이유는 뒤에 있는 `@` 때문이다. 

`@`가 확장 속성이 있음을 알리는 문자다. (혹은 `+`) 보통은 확장 속성이 있더라도 문제가 없지만, 특정 속성이 문제를 일으키는 것 같다. 전부 삭제해도 키를 사용함에는 문제가 없기 때문에 전부 지워준다.

확장 속석을 확인하기 위해 다음의 명령어를 사용한다. 그러면 다음과 같이 확장 속성 리스트가 표시된다.
```bash
xattr 파일명
com.apple.macl
com.apple.metadata:kMDItemWhereFroms
com.apple.provenance
com.apple.quarantine
```

파일에 있는 특정 확장 속성을 삭제하려면 다음과 같이 명령어를 사용하면 된다.
```bash
xattr -d 속성명 파일명

xattr -d com.apple.quarantine 파일명 # 예시
```

만약 전체 속성을 삭제하려면 다음의 명령어를 사용한다.
```bash
xattr -c 파일명
```