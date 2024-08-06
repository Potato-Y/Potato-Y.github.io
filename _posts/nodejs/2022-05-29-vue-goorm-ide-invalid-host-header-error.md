---
title: "[Vue.js + goorm ide] Invalid Host header"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Vue.js
---

구름에서 '항상 켜두기' 옵션을 무료로 오픈했다. 본래는 유료였지만 무료로 바뀐 만큼 활용도가 더욱 높아졌다.
마침 테스트용 프로젝트를 항시 오픈해야 하는 일이 있어서 타이밍이 좋았는데, 문제는 `npm run serve`를 하면 Invalid Host header 이란 문구와 함께 접근할 수가 없다.

구글에 검색하면 간단히 해결 방법이 나오는데 이는 지금 사용할 수 없다.

때문에 다음과 같이 수정해 주면 된다.

```js
module.exports = defineConfig({
  devServer: {
	allowedHosts: "all",
  },
});

```