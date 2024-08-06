---
title: "[electron + vue] build 결과물에서 router 작동하지 않는 문제 수정"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Vue.js
    - Electron
---

electron을 개발하면서 디버깅할 때는 `router`가 문제없이 작동하였는데, 이상하게 `build`만 하면 프로그램이 작동을 멈추었다.

이상하다 싶어서 보니 헤더 부분은 잘 나오는데 `router` 영역만 잘 나오지 않는다. 

이에 대한 내용이 https://xshine.tistory.com/345 이 블로그에 잘 정리되어 있다.

`index.js`
```js
import { createRouter, createWebHashHistory } from 'vue-router';
import HomeView from '../views/Home/HomeView.vue';

const routes = [
  {
    path: '/',
    name: 'home',
    component: HomeView,
  },
  {
    path: '/about',
    name: 'about',
    // route level code-splitting
    // this generates a separate chunk (about.[hash].js) for this route
    // which is lazy-loaded when the route is visited.
    component: () => import(/* webpackChunkName: "about" */ '../views/AboutView.vue'),
  },
];

const router = createRouter({
  history: createWebHashHistory(process.env.BASE_URL),
  routes,
});

export default router;
```

`history`를 `createWebHashHistory()` 메서드로 지정해 주면 된다.