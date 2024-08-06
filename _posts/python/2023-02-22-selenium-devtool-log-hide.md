---
title: "Selenium 개발도구 로그 숨기기"

toc: true
toc_sticky: true

categories:
    - Python
tags:
    - Python
    - Selenium
---
Selenium을 사용하다 보면 콘솔에 계속 오류가 표시된다. 그럼에도 동작에는 문제가 없기 때문에 해당 코드를 통해 로그 표시를 숨겨준다.

```py
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

# selenium을 위해 옵션 생성
options = Options()
options.add_experimental_option('excludeSwitches', ['enable-logging'])

browser = webdriver.Chrome(options=options)
```

