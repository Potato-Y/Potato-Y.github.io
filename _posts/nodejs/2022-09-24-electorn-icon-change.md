---
title: "Electron icon 변경하기"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Electron
---

`Electron`을 통해서 데스크톱 프로그램을 개발하던 중 icon을 변경해야 하는 시점이 왔다.

# Electron icon 변경
## 아이콘 만들기
`electron`에 적용할 아이콘을 만든다. 필자의 경우 [pixlr](https://pixlr.com/kr/)에서 `png` 확장명으로 만든 후 변환하였다. 아이콘을 만들 때에는 `1024x1024`의 크기를 베이스로 만든다. 최소 `512x512`가 권장인듯 하다. 그리고 추후에 수정 가능한 원본 파일을 꼭 만들어두자. 언제 수정이 필요할지 모른다. 필자의 경우 원본 파일인 `.pzx` 파일을 저장하였다.


### 아이콘 변환
아래의 웹 사이트에서 원하는 확장자 명 혹은 크기를 변환한다.
> PNG to ICO, ICONS
[https://cloudconvert.com/](https://cloudconvert.com/)

~~`icons` 파일은 Mac에서 사용하는 icon 확장명이다. 자동으로 사이즈가 resize 되지 않기 때문에 직접 특정 크기로 제작하여야 한다. ([참고](https://img2icnsapp.com/how-to-create-the-best-mac-icons/))~~

> 그런데 최신 버전에서는 `png` 파일만 있어도 된다. `electron-builder`에 자동으로 변환하는 기능에 문제가 있어 사용자가 수동으로 지정했는데, 현재는 이런 문제가 해결된 것으로 보인다. 

16px × 16px
32px × 32px
128px × 128px
256px × 256px
512px × 512px
1024px × 1024px

([참고](https://aroma-dev.tistory.com/2))

필자의 경우에는 위의 강조된 내용을 이유로 png 확장명으로 `1024x1024`, `512x512`를 준비했다. 

## 프로젝트 아이콘 변경
프로젝트 최상단에 `build/icons` 디렉토리를 만들고 밑의 사진처럼 `.png` 파일을 추가한다.
![image](https://github.com/user-attachments/assets/81a4f8d2-88ba-412c-b415-f390efea7d0a)

최소 `512x512` 이상의 크기로 구성해야 하는 것 같아서 2개의 파일을 추가하였다.

그런 뒤 빌드하면 아이콘이 정상적으로 잘 표시된다.