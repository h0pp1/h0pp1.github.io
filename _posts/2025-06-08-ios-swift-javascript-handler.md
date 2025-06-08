---
title: 'Swift에서 JavaScript를 다루는 방식'
author: hoppi
date: 2025-06-08 18:00:00 + 0900
categories: [Mobile]
tags: [mobile, iOS, Swift]
---
보통 안드로이드에서는 JavaScript에서 직접 `@JavascriptInterface` 어노테이션이 붙은 Java의 함수를 호출하는 형태로 앱과 상호작용합니다. 그렇다면 iOS의 Swift에서는 어떤 방식으로 상호작용 할 수 있을까요?

# JavaScript Handler 사용하는 부분 찾기
보통 iOS Swift에서 JavaScript와 통신하기 위해서는 `WKWebView`를 사용하고, JavaScript와 Swift 사이의 인터페이스를 설정하려면 `WKScriptMessageHandler`를 활용합니다. 안드로이드처럼 함수를 호출하는 형태가 아니라, 메시지를 주고 받는 식으로 웹과 앱이 상호작용합니다. 안드로이드와 간단하게 비교하면 아래와 같습니다.
| Android (WebView)              | iOS (WKWebView)                                  |
|-------------------------------|--------------------------------------------------|
| `@JavascriptInterface`         | `WKScriptMessageHandler` 프로토콜 구현           |
| `webView.addJavascriptInterface(...)` | `userContentController.add(_:name:)`      |
|  `InterfaceName.FunctionName()`        | `window.webkit.messageHandlers.ListenerName.postMessage(...)` |


보통 아래와 같이 `WKUserContentController.add()`를 통해서 Listener의 이름을 지정합니다.
```swift
private let contentController = WKUserContentController()
contentController.add(self, name: "iosListener")
```

그러면 JavaScript 부분에서 `window.webkit.messageHandlers.ListenerName.postMessage(data)`와 같이 사용합니다. 
```html
<!DOCTYPE html>
<html>
<head>
    <title>Test</title>
</head>
<body>
    <button onclick="sendMessage()">Send Message to iOS</button>

    <script>
        function sendMessage() {
            window.webkit.messageHandlers.iosListener.postMessage("Hello iOS");
        }
    </script>
</body>
</html>
```

메시지를 수신하여 처리하는 부분은 `WKScriptMessageHandler` 프로토콜을 이용해서 `userContentController()`를 구현합니다. 따라서 JavaScript로 전달된 데이터를 처리하는 로직을 확인하려면 `userContentController()`를 확인하면 됩니다. (프로토콜은 Java에서 인터페이스와 같은 개념입니다.)
```swift
func userContentController(_ userContentController: WKUserContentController,
                            didReceive message: WKScriptMessage) {
    print("Received Message: \(message.body)")
}
```
경우에 따라 `webView.evaluateJavaScript()`를 이용해 JavaScript 코드를 실행할 때, 사용자 입력값을 제대로 검증하지 않으면 XSS가 발생할 수도 있을 것 같습니다. 바이너리 대신 Swift 코드를 직접 보게 되는 경우는 많지 않을 것 같지만, 기록용으로 남겨둡니다.