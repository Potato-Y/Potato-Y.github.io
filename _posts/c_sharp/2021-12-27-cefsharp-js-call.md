---
title: "[C#] cefsharp JavaScript 호출"

toc: true
toc_sticky: true

categories:
    - C_Sharp
tags:
    - C#
    - JavaScript
---

개인적으로 만들고 있는 프로젝트가 있는데, 아무리 생각을 해도 winform만 가지고 디자인을 하면 원하는 스타일을 구현하는 데 한계가 있다고 판단했다.
기존 WebBrowser는 익스플로러 기반일 듯싶어 빠르게 chromium을 찾았다

###### ~~솔직히 레지스트리를 수정하는 게 주요 기능만 아니었으면 winform을 굳이 사용하진 않았을 듯..~~

# CefSharp
[CefSharp](https://github.com/cefsharp/CefSharp/)은 .NET에서 chromium을 사용할 수 있게 해주는 라이브러리다.

### winform에 컨트롤러 추가
```cs
private void Form1_Load(object sender, EventArgs e)
{
    this.browser = new ChromiumWebBrowser("http://127.0.0.1:5500/index.html");
    this.panel1.Controls.Add(browser);
}
```
## JavaScript 호출
### 기본적인 호출
[CefSharp WiKi에서 보기](https://github.com/cefsharp/CefSharp/wiki/General-Usage#1-how-do-you-call-a-javascript-method-from-net)
CefSharp에서 제공하는 위키에서는 다음 메서드를 통해 설명한다. >`browser.ExecuteScriptAsync`
그러나 실제로 사용해 보면 표면적으로 작동하지 않는다.

[참고 게시글](https://www.ostack.cn/?qa=65641/)을 보면 `문서에서 이미 언급했듯이 를 사용 ExecuteScriptAsync하면 스크립트가 비동기적으로 실행되므로 스크립트가 실제로 실행되기 전에 메서드가 반환됩니다.` 라고 한다.

###### ~~영어 실력 이슈~~로 제대로 확인하지 못해 이런 뻘짓이 일어났나 보다..

### 호출하기
```cs
private async void button1_Click(object sender, EventArgs e)
{
    this.browser.ExecuteScriptAsync("test()");
}
```
`ExecuteScriptAsync()`를 사용하면 정상적으로 호출되어 작동하는 것을 확인할 수 있다.