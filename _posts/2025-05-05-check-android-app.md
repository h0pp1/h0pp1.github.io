---
title: '안드로이드 앱을 점검할 때 어딜 봐야 할까?'
author: hoppi
date: 2025-05-05 16:00:00 + 0900
categories: [Mobile]
tags: [mobile, android]
---

오늘은 안드로이드 앱을 점검할 때 필수적으로 봐야할 곳에 대해서 이야기해보겠습니다.

크게 4가지를 생각할 수 있는데요. 아래와 같습니다.

- 웹뷰 핸들러
- 딥링크
- 자바스크립트 인터페이스
- 저장소

간략하게 요소마다 어떤 부분을 중점적으로 봐야 할지 알아보도록 하죠.

# 웹뷰(WebView)

## 웹뷰를 왜 사용할까?

---

일단 웹뷰는 앱 내에서 웹 콘텐츠를 표시하여 보여주는 View를 담당 합니다. 우리가 자주 사용하는 크롬이나 파폭과 같은 브라우저를 띄우는 것은 아닙니다. 그렇다면 웹뷰를 왜 사용할까요? 일단 첫 번째로, 앱을 다시 배포하는 수고를 덜 수 있습니다. 즉 앱을 업데이트가 필요 없다는 뜻입니다. 왜냐하면 웹 컨텐츠만 바꿔주면 웹뷰는 그것을 보여주는 역할을 하기때문입니다.

두 번째는 웹뷰는 하이브리드 앱에 필수적인 요소입니다. 요즘은 React Native나 Flutter 등과 같은 프레임워크를 통해 웹 기술로 앱을 개발하기 때문에 웹뷰가 기반이 됩니다.

마지막은 자바스크립트를 사용하기 위해서입니다. 웹 기능과 네이티브 앱을 결합하여 더 유연한 앱 환경을 만들 수 있는데, 이러기 위해서는 웹뷰가 필요합니다. 오늘날의 웹 콘텐츠에서 대부분 자바스크립트를 이용하는데, 웹뷰에서 설정만 해주면 자바스크립트를 쉽게 사용할 수 있습니다. (자바스크립트에 관한 내용은 뒤에서 더 자세하게 다루겠습니다)

## 웹뷰 사용 예시

---

간략한 웹뷰 설정은 아래와 같습니다. XML에 정의된 UI 요소를 `findViewById()`로 가져옵니다. 그다음 WebViewClient 객체를 정의하고 자바스크립트를 허용하여 `loadUrl()`을 통해서 URL을 로드합니다.

```xml
<WebView
    android:id="@+id/webview"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

```kotlin
import android.os.Bundle
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 1. WebView 객체 가져오기
        val webView = findViewById<WebView>(R.id.webview)

        // 2. 외부 브라우저가 아닌 앱 내에서 열리도록 설정
        webView.webViewClient = WebViewClient()

        // 3. JavaScript 허용
        webView.settings.javaScriptEnabled = true

        // 4. 웹페이지 로드
        webView.loadUrl("https://www.example.com")
    }
}

```

## 웹뷰 주요 설정

---

아래는 웹뷰 핸들러를 구성할 때, 자주 사용되는 설정들입니다.

| **메서드** | **설명** | **기본값** | **예시** | **보안 고려** |
| --- | --- | --- | --- | --- |
| **`setJavaScriptEnabled()`** | WebView에서 JavaScript 실행을 허용. | false | JavaScript가 필요한 페이지 렌더링 시 필요. | 신뢰할 수 없는 사이트에서 활성화하면 XSS 공격 위험. |
| **`setAllowFileAccess()`** | WebView 자체가 파일 시스템의 리소스에 접근하도록 허용. 즉, file://을 사용하는 것 (API Level 30 이하는 default가 true) | false | 로컬 HTML, CSS, JS 파일을 로드할 때 사용. | 파일 시스템 접근은 민감한 데이터 노출 위험이 있으므로 필요 시에만 활성화. |
| **`setAllowFileAccessFromFileURLs()`** | 로드된 HTML에서 file://을 이용하여 로컬 파일을 로드하는 것을 허용. (API Level 30 부터 deprecated 됨.) | false | 로컬 HTML이 다른 파일(이미지 등)을 참조 시. | 다른 로컬 리소스 접근 허용 시 디렉토리 트래버설 공격 가능성. 필요하지 않다면 비활성화. |
| `setDomStorageEnabled()` | localStorage, sessionStorage 허용. | false | localStorage를 사용할 시. | 저장된 데이터 탈취 위험. 민감 정보 저장 피해야 함. |

****** `setAllowUniversalAccessFromFileURLs()`는 **`setAllowFileAccessFromFileURLs()`와 다르게** 다른 외부 리소스(http://, https://)에 접근할 수 있음.*

## 점검 포인트

---

웹뷰 핸들러로 의도하지 않은 URL을 전달할 수 있는지 확인해야 합니다. 만약 잘못된 URL 호스트 검증 방식을 사용한다면, 악의적인 URL을 로드시킬 수 있기 때문입니다. 또한, 개발자 입장에서는 안전한 URL 파싱 라이브러리를 이용하여 scheme과 host를 체크할 수 있도록 합니다. host의 경우는 Allowlist를 이용하여 파싱된 호스트가 리스트에 존재하는지, 서브 도메인을 체크할 때는 `.example.com`  으로 끝나는지 `endsWith()`으로 검증하는 게 안전합니다.

# 딥링크(Deeplink)

---

딥링크는 일반적인 웹 URL과 유사하지만, 주소를 입력하면 특정 앱이나 화면을 실행시키는 기술입니다. 다음과 같이 *AndroidManifest.xml*에서 설정할 수 있습니다.

- `android:exported="true"`: 외부 앱에서도 이 액티비티를 호출 가능하게 함. (명시적으로 설정 필요)
- `android.intent.action.VIEW`: 딥링크 URL로 액티비티를 실행하도록 함.
- `android:scheme="test"` + `host="deeplink"`: `test://deeplink/path` 로 호출 가능.

```xml
<activity android:name=".MainActivity"
    android:exported="true">

    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="test" android:host="deeplink" />
    </intent-filter>

</activity>
```

또한 아래와 같이 `adb` 명령어를 통해서 쉽게 설치되어있는 특정 앱의 액티비티를 실행할 수 있습니다.

```bash
adb shell am start -a android.intent.action.VIEW -d "test://deeplink"
```

## 딥링크 핸들링하는 코드 예시

---

코드단에서 어떻게 딥링크를 핸들링하는지 간단한 예시로 살펴봅시다. 이용자가 `test://deeplink?url=http://example.com`  을 클릭하면, `onCreate()` 가 실행되고 인텐트의 데이터를 파싱합니다. 그러면 `getQueryParamemter()`를 통해서 `url` 파라미터의 값을 파싱하여 WebViewActivity로 `http://example.com`을 인텐트에 담아서 전달합니다. 

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // 1. 인텐트에서 URI 데이터 추출
        val data: Uri? = intent?.data

        if (data != null) {
            // 2. 쿼리 파라미터 "url" 추출
            val url = data.getQueryParameter("url")

            if (!url.isNullOrEmpty()) {
                // 3. WebViewActivity로 이동
                val intent = Intent(this, WebViewActivity::class.java)
                // 4. 추출한 URL을 인텐트에 담아서 전달
                intent.putExtra("url", url)
                startActivity(intent)
            }
        }
    }
}
```

## 점검 포인트

---

딥링크의 `host`와 `scheme`에 대해 검증하고 있는지 체크해야 하고, 딥링크에서 사용하는 파라미터에 대한 검증이 필요합니다. 예를 들어, `url` 파라미터처럼 원래는 의도된 URL만 전달해야 하는데, 해당 값에 대한 검증이 없으면 공격자가 원하는 URL을 전달할 수 있습니다. 그러면 웹뷰에 공격자의 사이트를 웹뷰에 띄어서 자바스크립트 인터페이스와 연계하는 공격이 가능해집니다.

# 자바스크립트 인터페이스(JavascriptInterface)

---

자바스크립트 인터페이스는 웹과 네이티브(Android, iOS) 앱 간의 데이터 및 기능을 상호 호출할 수 있도록 하는 브릿지 역할을 합니다. 안드로이드에서는 웹뷰를 설정할 때, **`addJavascriptInterface()`**를 이용하여 설정할 수 있습니다.

## 자바스크립트 인터페이스 예시

---

아래와 같이 웹뷰를 설정하는 부분에서 `addJavascriptInterace()`를 통해서 사용될 객체와 이름을 설정합니다.

```kotlin
webView.addJavascriptInterface(JSBridge(this), "Android")
```

 JSBridge에서는 `@JavascriptInterface` 어노테이션을 사용하여 자바스크립트에서 사용할 수 있는 함수를 정의합니다.

```kotlin
class JSBridge(val context: Context) {

    @JavascriptInterface
    fun sendSms(number: String, text: String) {
        val intent = Intent(Intent.ACTION_SENDTO, Uri.parse("smsto:$number"))
        intent.putExtra("sms_body", text)
        context.startActivity(intent)
    }
}

```

만약, 위에서 설명한대로 악성 웹 사이트가 웹뷰에 로드된다면, 이용자의 핸드폰 번호로 아래와 같이 설정된 함수를 악용할 수 있습니다.

```html

<script>
  Android.sendSms("01012345678", "모바일 청첩장 링크입니다: https://bit.ly/Eaxpq");
</script>

```

## 점검 포인트

---

악의적인 사이트를 웹뷰에 로드할 수 있을 때, 이용자의 민감한 정보를 뿌려주는 자바스크립트 인터페이스 함수가 있는지 체크해야 하고, 함수에서 전달받은 파라미터에 대한 검증이 적절하게 이루어지고 있는지 등을 확인해야 합니다.

# 저장소(Storage)

---

안드로이드에서 사용되는 저장소들 중에 가장 많이 보는 것이 `SharedPreferences`와 `SQLite`라고 생각됩니다. 그 중에서 SharedPreferences에 대해서 간략하게 적어보고자 합니다.

SharedPreferences는 안드로이드에서 데이터를 key-value 쌍으로 로컬(디바이스)에 저장할 수 있게 해주는 저장소 역할을 합니다. 저장 위치는 `/data/data/<package_name>/*.xml`와  같은 형태로 저장됩니다. 주로 사용자 설정값처럼 민감하지 않은 데이터를 저장하는 데 사용됩니다.

> [안드로이드 개발자 문서](https://developer.android.com/training/data-storage/shared-preferences?hl=ko)에 따르면, 지금은 `DataStore`를 사용하는 것을 권장하고 있습니다. 또한 [EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences)도 현재는 deprecated 되었습니다.
{: .prompt-info } 

## 점검 포인트

---

SharedPreferences에 세션 토큰, 로그인 정보 등과 같은 민감한 정보를 포함할 때가 있습니다. 물론, 앱 내부 저장소에 파일이 위치하여 루팅 되지 않은 디바이스에서는 접근이 어렵지만, 그래도 평문으로 저장한다면 다른 위협을 통해서 데이터를 유출 시킬 가능성은 항상 존재하게 됩니다.