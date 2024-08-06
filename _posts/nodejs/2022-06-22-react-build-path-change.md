---
title: "[React] 빌드 폴더 변경하기"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Express
    - React
---

vue와 express를 하나로 통합해서 web을 표시하는 서버와 api서버를 하나로 합쳐서 제작해본 적이 있다. (설명이 이상한데.. frontend와 backend를 하나로 통합) vue에서는 vue.config.js를 [수정](https://github.com/Potato-Y/simple-questionnaire/blob/main/frontend/vue.config.js)해서 `backend/public` 폴더에 빌드가 되도록 설정했었다. react에서도 동일하게 설정하고자 해서 설정법을 찾아보았었는데, react는 vue하고 다르게 설정하는 것을 보았다. ~~(vue에 비해 편하다면 편한..?)~~

## React package.json 편집
Vue와는 다르게 `package.json` 파일을 편집해준다.
```json

  "scripts": {
    "start": "react-scripts start",
    "build": "BUILD_PATH='../backend/public' react-scripts build",
    
    ...
    
  }

```
`BUILD_PATH='../backend/public'`와 같이 폴더명을 명시해주면 된다.

### 윈도우에서 사용하기
윈도우에서는 위의 스크립트 그대로 사용하면 오류가 발생한다.
```js
  "scripts": {
    "wbuild": "set \"BUILD_PATH=..\\backend\\public\" && react-scripts build",
      ...
  }
```
