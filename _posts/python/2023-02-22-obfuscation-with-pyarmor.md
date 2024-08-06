---
title: "pyarmor로 난독화 후 배포를 위한 파일 만들기"

toc: true
toc_sticky: true

categories:
    - Python
tags:
    - Python
    - pyarmor
---

마침 python을 공부하고 있던 중, 특정 요구 사항이 들어간 프로그램 제작 요청이 들어왔다. C#으로 제작된 프로젝트는 배포를 해본 적이 있지만, python은 처음이라 이리저리 찾아보니 좋은 모듈이 있었다.

> 다만 이런 문제가 있을 수 있다. 본인이 제작한 프로그램도 이러한 문제가 발생하여 백신 프로그램에 의해 삭제되었다.
만약 백신에 검사당하기를 원치 않는다면, 난독화를 포기해야 할 듯싶다. ~~(사용자들에게 백신에서 제외하라고 말하기에는 신뢰가 아직 부족하다면..)~~


먼저 필요한 패키지를 설치해 준다.
```bash
pip install pyarmor
pip install pyinstaller
```

그 후에 터미널의 위치를 main.py가 있는 곳으로 이동한다.

그런 다음 아래의 명령어를 통해 난독화 및 실행 파일을 만들어준다.

```bash
$ pyarmor pack --clean -e "--onefile " main.py
```

작업이 완료된 다음에는 `dist` 폴더에 파일이 생성된다.

옵션은 `pyinstaller`에서 지원하는 내용들로 채우면 되는 것 같다.
만약 콘솔이 표시되지 않게 하려면 아래와 같이 하면 된다. 

```bash
pyarmor pack --clean -e "--noconsole --onefile " main.py
```

그리고, 아이콘을 추가하고 싶다면 아래와 같이 하면 된다.

```bash
pyarmor pack --clean -e "--onefile --icon=icon.ico" main.py
```
필자의 경우에는 `icon` 파일의 위치를 `main.py`와 동일하게 해주었다. `icon`을 설정하는 방법에 대해 찾아보니 절대 위치를 입력하라는 얘기도 있었지만, 상대 위치로 설정하니 정상 작동하였다. 아마 `pyarmor`에서 입력된 상대 위치를 절대 위치로 변환하는 것 같다.

무료 아이콘: https://icon-icons.com/ko/