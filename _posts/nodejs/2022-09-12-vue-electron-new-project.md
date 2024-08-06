---
title: "Vue.js + Electron 새로운 프로젝트 만들기"

toc: true
toc_sticky: true

categories:
    - Node.js
tags:
    - Node.js
    - Electron
    - Vue.js
---

Electron의 프론트앤드를 만들 땐 여러 가지 방법이 있다. 제일 많이 알고 있는 방법인 React를 사용하는 방법과 Vue를 사용하는 방법이 있다. 본인의 경우 React로 구성할 경우 초반 작업이 귀찮고, 이전에 다뤄본 Vue가 편했던 느낌이 남아있어서 Vue를 사용하기로 했다.

# Vue + Electron
## Vue CLI 설치
`Vue Cli`를 전역으로 설치한다. (혹시 `@vue/cli` 이전 버전이 설치되어 있다면 삭제 후 설치한다.)

```bash
$ npm i -g @vue/cli
```
설치를 완료한 뒤 설치 버전을 확인한다.

```bash
$ vue --version
@vue/cli 5.0.8
```

## Vue 프로젝트 생성
새로운 `vue` 프로젝트를 생성한다.
```bash
$ vue create elec-vue
```
![image](https://github.com/user-attachments/assets/c03ee439-c923-4880-90b0-f877f51cb22b)
![image](https://github.com/user-attachments/assets/6cb6fb6f-3888-404e-96ce-0f00304f4c30)
필요한 옵션을 입력한다.
![image](https://github.com/user-attachments/assets/71b9f987-ff40-40c7-97ab-1a24c2731d09)

이어서 프로젝트 생성 단계를 진행한다.

생성을 완료한 뒤, 프로젝트를 실행한다.
```bash
$ cd elec-vue
$ npm run serve
##	or  ##
$ yarn serve
```

잘 실행되는 것을 확인하고 종료한다.

## Electron Builder 추가
`Electron Builder`는 기존 프로젝트를 `Electron`으로 바꾸고 쉽게 `build` 할 수 있게 도와준다.
`Vue CLI Plugin`으로 제공하는 `Electron`을 사용하면 쉽게 설치할 수 있다.
[Vue CLI Plugin Electron Builder](https://nklayman.github.io/vue-cli-plugin-electron-builder/)

명령어로 `Electron Builder` 추가한다.
```bash
$ vue add electron-builder
```
![image](https://github.com/user-attachments/assets/8cbf7bc0-7f24-468a-95a9-56f41ad90848)

현재 최신 버전인 `13.0.0`을 선택한다.

설치가 완료되면 `package.json`에 스크립트가 추가된다.

## Electron 실행
이제 프로젝트를 Electron으로 실행한다. 
```bash
$ npm run electron:serve
##  or  ##
$ yarn electron:serve
```

![image](https://github.com/user-attachments/assets/0d1c4b57-328c-4fd8-a091-24a726e1526f)

정상적으로 실행되는 것을 확인할 수 있다.

[참고](https://codegear.tistory.com/83)
