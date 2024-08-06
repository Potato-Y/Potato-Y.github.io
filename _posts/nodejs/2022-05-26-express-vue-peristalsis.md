---
title: "[Express+Vue.js] Express로 frontend와 backend 연동하기"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Express
    - Vue.js
---

Vue.js 프로젝트를 진행하면서 하나의 Express 프로젝트에서 frontend와 backend를 모두 실행하고자 했다. 역시 구글에 찾아보니 방법이 있었다.

## 프로젝트 폴더 만들기
### express 프로젝트 생성
``` bash
$ npm install -g express-generator
$ express --view=pug backend
```
npm으로 [express-generator](https://www.npmjs.com/package/express-generator)를 설치한다.
`backend` 이름으로 새 express 프로젝트를 간단하게 생성한다.

backend express 프로젝트로 이동하여 `npm i`를 진행한다.

### vue 프로젝트 생성
```bash
$ npm install -g @vue/cli
$ vue init webpack frontend
```
npm으로 [@vue/cli](https://cli.vuejs.org/)를 설치해준다.

## Vue 프로젝트 수정
vue 프로젝트 설정을 수정한다.

`vue.config.js`
```js
const { defineConfig } = require('@vue/cli-service');
const path = require('path');

module.exports = defineConfig({
  outputDir: path.resolve(__dirname, '../backend/public/'),
  transpileDependencies: true,
  devServer: {
    proxy: {
      '/api': {
        target: 'https://127.0.0.1:3000/api',
        changeOrigin: true,
        pathRewrite: {
          '^/api': '',
        },
      },
    },
  },
});

```
api를 정상으로 불러올 수 있도록 `proxy`를 설정한다.

## Express 프로젝트 수정
### connect-hisotry-api-fallback
Vue 에서 사용하는 router가 정상적으로 작동할 수 있도록 [connect-history-api-fallback](https://www.npmjs.com/package/connect-history-api-fallback)을 추가해준다.
```bash
$ npm i connect-history-api-fallback
```

`app.js`를 확인하면 `express`에 추가한 `router`가 다음과 같이 되어있다.
```js
app.use('/', indexRouter);
app.use('/users', usersRouter);
```
아래와 같이 위 코드를 수정해준다.
```js
const history = require('connect-history-api-fallback');

	...
    
app.use('/users', usersRouter);
app.use(history()); // vue router 기능을 위해 추가
app.use('/', indexRouter);
```

api로 사용할 라우터를 `indexRouter`보다 위에 위치해준다. Express를 REST API 서버로 함께 사용하면서 GET Request를 사용하기 때문에 `history()`의 위치가 중요하다. Express는 위에서 부터 미들웨어가 차례로 실행되기 때문이다.

~~그럼에도 vue router에서 설정한 링크로 들어가면 404에러가 어김없이 나온다. 깃허브에 올린 페이지도 그런 것 보면 필자의 능력부족인 듯 싶다...~~

### router 수정
`index` router를 수정해준다.
```js
var express = require('express');
var router = express.Router();
var path = require('path');

/* GET home page. */
router.get('/t', function (req, res, next) {
  res.sendFile(path.join(__dirname, '../public/index.html'));

  // res.render('index', { title: 'Express' }); // 기존 파일
});

module.exports = router;
```

## 간단한 스크립트 추가
한번에 frontend와 backend를 빌드하기 위해 각 프로젝트별로 스크립트를 추가했다. 
~~아마 `package.json`에 스크립트를 추가하면 될 것 같은데.. 급한대로 스크립트를 따로 만들었다...~~

| backend/script.sh
```bash
#!/bin/bash

# Vue 프로젝트 빌드
echo "Start building your VUE project"

cd ../frontend
npm run build

# express 시작
echo "Start Backend"
cd ../backend
npm start

exit 0
```

| frontend/script.sh
```bash
#!/bin/bash

# Vue 프로젝트 빌드
echo "Start building your VUE project"

npm run build

# express 시작
echo "Start Backend"
cd ../backend
npm start

exit 0
```


참고한 글 https://yorr.tistory.com/6?category=753153

