---
title: "Vue + Electron builder 문제 해결"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Vue.js
    - Electron
---

`electron` 프로젝트를 만지던 중 아이콘 적용을 위해 빌드를 시도했는데.. 이럴 수가.. 오류를 내뿜는다. 대충 오류의 내용은 아래와 같다.
```error
Error: Exit code: ENOENT. spawn /usr/bin/python ENOENT
...
error Command failed with exit code 1.
```

Python3가 이미 설치되어 있더라도 Pyhton2를 불러오기 때문에 문제가 발생하여 Python2를 설치했으나 계속 실패하였다.

`electron-builder`의 문제로 이미 최신 버전에서 수정되었으나, `vue-cli-plugin-electron-builder`에서는 `electron-builder v22`를 사용하고 있었다.

이에 대한 문제는 [https://github.com/nklayman/vue-cli-plugin-electron-builder/issues/1701#issuecomment-1099369036](https://github.com/nklayman/vue-cli-plugin-electron-builder/issues/1701#issuecomment-1099369036) 이 곳에서 해결법을 제시하고 있다.

```js
"overrides": {
    "vue-cli-plugin-electron-builder": {
      "electron-builder": "^23.0.3"
    }
  }
```
위 내용을 `package.json`에 추가한다.

그리고 `node_modules`를 삭제하고 다시 `npm i`를 실행한 뒤에 `electron:build`를 시작하면 정상적으로 빌드가 가능하다.