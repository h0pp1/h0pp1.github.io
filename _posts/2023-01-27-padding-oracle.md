---
title: '[Crypto] Padding Oracle Attack'
author: hoppi
date: 2023-01-27 23:36:00 + 0000
categories: [Crypto]
tags: [crypto, padding_oracle, dreamhack]
---
카뱅 면접 볼 때 harry가 내 이전 블로그에 있던 드림핵 문제였던 [Padding Oracle](https://dreamhack.io/wargame/challenges/127/) 풀이를 보고 `패딩 오라클이 뭔가요?` 라고 질문 했는데 답변하지 못했다. 질문 폭격을 맞는 상황인 것도 있었지만 정말로 내가 풀었는데 정확하게 어떤 의미인지 기억을 못해서 답변을 못하는 어처구니 없는 상황이 발생한 것이다. 그래서 다시 봐야지를 오랫동안 실천하지 못(?)하다가 지금 적어본다.  


# Padding Oracle Attack이란?
***
오라클은 쉽게 말하면 어떤 시스템이라고 생각하면 된다. 그리고 패딩은 블록 암호화에서 사용하는 것인데 `여기는 빈칸인데요`를 알려주는 빈칸 채우기 용 값이다. 즉 `Padding Oracle Attack`은 <u>패딩이 올바른지 아닌지를 판단하는 과정을 악용하여 평문을 얻을 수 있는 공격방식이다.</u>  

# Example of Padding
***
말보다는 예시를 들면 더 잘 이해할 수 있을 것이다. 예를들어 블록의 사이즈가 16바이트고 문자열이 `hoppi`이면 아래 표와 같이 0xB가 11개 저장된다. (10글자이면 0x6이 6개, 7글자이면 0x9가 9개... 이런식이다.)  

|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|  
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|h|o|p|p|i|0xB|0xB|0xB|0xB|0xB|0xB|0xB|0xB|0xB|0xB|0xB|

<br/>
그렇다면 블록의 사이즈가 16인데 16글자라면?? 0x10으로 한 개의 블럭이 추가로 채워진다. 이렇듯 <u>마지막 블록에는 무조건 패딩 바이트가 존재한다.</u> (블록 사이즈가 16인데 0x15, 0xf1 이런 바이트는 올 수 없다는 뜻이다.)  

|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|  
|I|_|h|a|t|e|_|c|a|r|r|o|t|s|!|!|
|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|<span style="color:yellow">0x10</span>|

<br/>

# 이 문제가 Padding Oracle Attack이 가능했던 이유
***
## CBC의 특성
***
CBC의 특성을 살펴보면 암호화된 이전 블록과 평문 블록을 `XOR`하여 암호문 블록을 생성한다. 복호화할 때는 이 과정을 거꾸로 한다고 보면 된다. 이전 블록이 다음 블록 결과에 영향을 주기 때문에 암호화 시에는 병렬 처리가 불가능하지만 복호화 시에는 이전 블록만 올바르다면 병렬 처리가 가능하다. (초밥을 먹을 때처럼 먹고 싶은 거 먼저 골라 먹을 수 있다는 소리)  
![cbc](../../../assets/img/2023-01-27/cbc.png){: w="600" h="300" }  
<br/>

우리는 `n-1`번째 암호화 블록과 `n`번째 평문 블록을 `XOR`한 값(n 번째 블록을 암호화 하기 이전의 값 == Plaintext ⊕ Ciphertext)을 알 수 없다. 하지만 이 값만 알게 되면 아래와 같이 `XOR` 연산의 특성으로 평문 블록을 얻을 수 있다.  
![hand](../../../assets/img/2023-01-27/hand.png){: w="600" h="300" }  
<br/>

## 에러 메시지
***
그렇다면 어떻게 `n-1번째 암호화 블록`과 `n 번째 평문 블록`을 `XOR`한 값을 알 수 있을까? 패딩 오라클 공격을 수행하기 위해서는 하나의 조건이 필요한데 바로 패딩 오라클이 정상적으로 수행됐는지 아닌지를 판단하는 **서버의 응답**이다.  

분명 평문이 어떻게 끝나든 마지막 평문 블록안에는 패딩 바이트가 존재하게 되어있다. `n-1번째 암호화 블록`의 마지막 바이트의 값에 따라서 빨간색 동그라미와 `XOR`한 결과 값이 패딩 바이트면 시스템은 평문으로 인식할 것이다. 그것이 아니라면 어떠한 오류를 나타낼 것이다. 우리가 알고 있는 것은 `n-1 번째 암호화 블록`이다. 따라서 `n-1 번째 엄호화 블록`의 마지막 바이트를 0x0 ~ 0xff 까지 brute forcing을 하여 평문 블록의 마지막 바이트가 0x1이 나오는 `n-1 번째 엄호화 블록`의 마지막 바이트를 찾으면 자연스럽게 `n-1번째 암호화 블록`과 `n번째 평문 블록`을 `XOR`한 값(Plaintext ⊕ Ciphertext)의 마지막 바이트를 알 수 있게 된다. 이런식으로 시스템이 `패딩이 올바르네요!`라고 착각하게 만들어야 한다. 그렇기 때문에 `Plaintext ⊕ Ciphertext`의 15 번째 바이트를 구하려고 하면 `n 번째 평문 블록`의 15 번째 바이트를 0x2로 설정해줘야 할 뿐만 아니라 마지막 바이트도 0x2로 설정해야한다. (14 번째를 구할 때는 0x3, 0x3, 0x3 ...) 이렇게 첫 번째 바이트(0x10)까지 도달하면 `Plaintext ⊕ Ciphertext`의 모든 바이트를 구할 수 있다. 이런식으로 위 사진처럼 `Plaintext ⊕ Ciphertext`를 알았고 `n-1 번째 암호화 블록`도 알기 때문에 이 둘을 `XOR` 하면 평문 블록 또한 구할 수 있게 된다.  

아무튼 이 문제에서는 사용자가 관리자 그룹에 속해있는지와 비밀 문서를 작성한 사람이 본인인지 판단하기 전에 복호화를 하고 있고 또한 오류까지 친절하게 알려주고 있어서 `Padding Oracle Attack`이 가능했다.  
<br/>

내가 이해가 안된 부분을 위주로 글로 적은 것이라서 부분부분 이해가 안가는 부분이 있을 수 있다. 레퍼런스에 있는 글을 참고하면 더 자세히 알 수 있을 것이다.  

# Reference
- [https://hacksms.tistory.com/50](https://hacksms.tistory.com/50)

