---
title: "Vue + Electron IPC 통신"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Electron
    - Vue.js
---

Electron은 `Main Process`와 `Renderer Process`로 이루어져있다. 웹 페이지를 보여주기 위해 `Chromium`을 사용하기 때문에 `Chromium`의 멀티프로세스 아키텍쳐를 그대로 사용한다. Electron 안에서 보여지는 각각의 웹페이지는 자신의 프로세스 안에서 동작하는데, 이 프로세스가 `Renderer Process`이다. 일반적으로 브라우저에서 웹페이지는 보통 샌드박스 환경에서 실행되고 네이티브 리소스에는 엑세스 할 수 없다. 그렇기 때문에 `IPC` 통신을 통하여 `Node.js`의 `API`를 사용할 수 있다.
[참고 1](https://lifeinprogram.tistory.com/4)

### Vue config 수정
`vue.config.js`를 수정해야 한다. [이 곳을 참고](https://nklayman.github.io/vue-cli-plugin-electron-builder/guide/security.html#node-integration)

```js
module.exports = {
  pluginOptions: {
    electronBuilder: {
      nodeIntegration: true
    }
  }
}
```
혹은
```js
const { defineConfig } = require('@vue/cli-service');
module.exports = defineConfig({
  transpileDependencies: true,
  pluginOptions: {
    electronBuilder: {
      nodeIntegration: true,
    },
  },
});
```
위와 같이 수정한다.

### App.js 수정
`template`에 버튼을 추가하고 메소드를 연결한다.
```vue
<template>
  <nav>
    <router-link to="/">Home</router-link> |
    <router-link to="/about">About</router-link>
  </nav>
  <button @click="test">테스트</button>
  <router-view />
</template>
```

스크립트 부분을 아래와 같이 수정한다.
```js
export default {
  methods: {
    test() {
      const { ipcRenderer } = require("electron");
      console.log(ipcRenderer.sendSync("synchronous-message", "ping")); // "pong"이 출력됩니다.

      ipcRenderer.on("asynchronous-reply", (event, arg) => {
        console.log(arg); // "pong"이 출력됩니다.
      });
      ipcRenderer.send("asynchronous-message", "ping");
    },
  },
};
```

### background.js 수정
```js
'use strict';

import { app, protocol, BrowserWindow, ipcMain } from 'electron';

...
```
`ipcMain`을 추가한다.

```js
...

ipcMain.on('asynchronous-message', (event, arg) => {
  console.log(arg); // "ping" 출력
  event.reply('asynchronous-reply', 'pong');
});

ipcMain.on('synchronous-message', (event, arg) => {
  console.log(arg); // "ping" 출력
  event.returnValue = 'pong';
});
```

[참고](https://arikong.tistory.com/10)
### 실행
```bash
$ npm run electron:serve
## or ##
$ yarn electron:serve
```
프로젝트를 실행하면 아래와 같이 테스트 버튼이 정상적으로 작동하는 것을 확인할 수 있다.
![image](https://github.com/user-attachments/assets/43c47263-4955-4a53-a5be-3eecbab518a5)
