---
title: '[Mobile] APK 리패키징 & 서명 방법 / 예제'
author: hoppi
date: 2023-02-01 17:01:00 + 0000
categories: [Mobile]
tags: [mobile, apk_repackaging, apk_signing, smali, android_challenge]
---

# 리패키징을 위한 Smali 코드 변경
***
## Smali 코드란?
***
`smali`는 Dalvik VM(바이트코드를 해석하고 실행하는 가상머신)에서 쓰이는 `dex` 파일을 사람들이 쉽게 읽을 수 있도록하는 코드이다.  

## Smali 코드 변경
***
스말리 코드를 추출하기 위해서 [apktool](https://ibotpeaches.github.io/Apktool/install/)을 사용한다. 다음과 같은 명령어를 입력해준다.  
```zsh
apktool d <디컴파일할 apk>
```

![apktool](../../../assets/img/2023-02-01/apktool.png){: w="600" h="300" }  
<br/>

디컴파일된 폴더로 가보면 이렇게 나온다. 코드를 수정할 일이 있으면 smali 폴더로 가서 해당 클래스에 맞는 파일을 수정해주면 된다.  
![ls](../../../assets/img/2023-02-01/ls.png){: w="600" h="300" }  
<br/>

그리고 다시 apk파일로 압축하기 위해서 아래 명령어를 실행해준다.  
```zsh
apktool b <디컴파일된 폴더> -o <생성할 apk 이름>
```

![recompile](../../../assets/img/2023-02-01/recompile.png){: w="600" h="300" }  
<br/>

# APK 파일 서명
***
만약 리패키징을 하고나서 그대로 설치한다고하면 설치가 안된다. 자신이 개발했다는 것을 증명하기 위해 서명을 해야한다.  
먼저 `keytool` 명령어를 통해서 `keystore`을 만들어준다.  
```zsh
keytool -genkey -v -keystore <keystore 이름> -alias <alias 이름> -keyalg <알고리즘> -keysize <키사이즈>
```

![keytool](../../../assets/img/2023-02-01/keytool.png){: w="600" h="300" }  
<br/>

그러면 key파일이 생성되었을텐데 `jarsigner` 명령어를 통해서 apk파일에 서명을 해준다.  
```zsh
jarsigner -verbose -sigalg <서명 알고리즘> -digestalg <digest 알고리즘> -keystore <keystore 파일> <서명할 apk파일> <나의 alias 이름>
```

![jarsigner](../../../assets/img/2023-02-01/jarsigner.png){: w="800" h="400" }  
<br/>

# 예제 (NahamCon CTF 2022 Moblie Challenge - Click Me!)
***
## 분석
***
[Click Me!](https://github.com/evyatar9/Writeups/blob/master/CTFs/2022-NahamCon_CTF/Mobile/Click_Me/click_me.apk)라는 문제를 통해서 알아보자.  
apk를 폰에 설치하면 다음과 같은 화면이 나오는데 GET FLAG 버튼을 누르게 되면 `You do not have enough cookies to get the flag` 이라고 한다.  
![cookie](../../../assets/img/2023-02-01/cookie.png){: w="300" h="150" }  
<br/>

jadx를 통해서 디컴파일해보면 `getFlagButtonClick()`에서 클릭 횟수가 99999999일 때 `getFlag()`를 호출한다.
![getflag](../../../assets/img/2023-02-01/getflag.png){: w="700" h="350" }  
<br/>

만약 직접 누른다고 하더라도 아래 `cookieViewClick()`를 통해서 13371337번 이상 누르게되면 이 값으로 고정되기 때문에 플레그를 얻는 것은 불가능하다.  
![cookieclick](../../../assets/img/2023-02-01/getflag.png){: w="700" h="350" }  
<br/>

## Get Flag
***
여러 방법으로 풀 수 있지만 이 글의 주제인 리패키징을 통해서 해결해 보자.  
`this.CLICKS != 99999999` 일 때로 바꾸거나 조건문을 타지 않더라고 `getFlag()`를 Toast.makeText의 인자로 넣어줄 수 있다. (한번 둘 다 적용해 보자)  
<br/>

먼저 `this.CLICKS != 99999999`로 바꿔보자. 아래와 같은 스말리 코드로 변경한다. 처음에는 `if-ne`로 되어있었으므로 그 반대인 `if-eq`로 바꿔준다.  
![eq](../../../assets/img/2023-02-01/eq.png){: w="600" h="300" }  
<br/>

그다음 1번만 클릭해도 플레그를 볼 수 있도록 기존 `You do not have enough cookies to get the flag` 문자열이 아니라 `getFlag()`를 인자 값으로 넣어준다.  
.line 36과 비교해서 눈치껏 getFlag()를 .line 38 바로 아래에 넣어주고 기존 문자열을 아래와 같이 주석처리 하였다.  
![36](../../../assets/img/2023-02-01/36.png){: w="700" h="350" }  
<br/>


다시 리패키징을 해주고 jadx로 열어보면 아래와 같이 코드수정이 된 모습을 볼 수 있다.  
![edit](../../../assets/img/2023-02-01/edit.png){: w="700" h="350" }  
<br/>

그리고 apk 서명을 해주고 다시 설치해주면 1번만 클릭해도 플레그를 얻을 수 있다.  
<br/>

# Reference
***
- [https://domdom.tistory.com/287](https://domdom.tistory.com/287)
