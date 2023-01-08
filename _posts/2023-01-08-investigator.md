---
title: '[Hackthebox] Investigator'
author: hoppi
date: 2023-01-08 23:44:00 + 0000
categories: [Mobile]
tags: [mobile, forensic, hackthebox]
---

**키워드: Mobile, Forensic**  

![description](../../../assets/img/2023-01-08/2023-01-08-description.png){: w="600" h="300" }  
<br/>

# Crack password
***
문제 파일을 압축해제하면 backup.ab 파일과 system 폴더가 주어진다. 확장자 ab는 `android backup file`을 뜻한다. 이것에 대해서 조사를 해봤을 때 `adb restore` 명령어로 복구가 가능하다고 해서 폰을 연결하고 해봤더니 비밀번호가 걸려있는지 정상적으로 풀어지지 않았다.  
![restore](../../../assets/img/2023-01-08/2023-01-08-restore.png){: w="200" h="100" }  
<br/>

문제에서도 말했듯이 패스워드를 찾아 풀어야하는 것 같다. system 폴더에 들어가보면 .key 파일들이 보인다. 
![key](../../../assets/img/2023-01-08/2023-01-08-key.png){: w="600" h="300" }  
<br/>

[여기](https://blog.naver.com/PostView.naver?blogId=sjhmc9695&logNo=222420204223&parentCategoryNo=&categoryNo=&viewDate=&isShowPopularPosts=false&from=postView)와 [이곳](https://www.pentestpartners.com/security-blog/cracking-android-passwords-a-how-to/)을 참고해서 분석해본 결과 아래와 같이 정리할 수 있다.  
먼저 device_policies.xml 파일에서 어떤 유형의 패스워드를 사용하는지 찾을 수 있다. 아래와 같이 5글자의 소문자로 구성되어있는 것을 알 수 있다.  
```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<policies setup-complete="true">
<active-password quality="262144" length="5" uppercase="0" lowercase="5" letters="5" numeric="0" symbols="0" nonletter="0" />
</policies>
```
{: file="device_policies.xml"}  
<br/>

이렇게 되면 gesture.key 파일은 필요가 없다. <u>gesture 파일 같은 경우는 패턴을 설정하면 생기는 key 파일로 현재는 패스워드를 사용하고 있기 떄문에 password.key 파일을 분석하면 된다.</u>  

`password.key` 파일은 총 72바이트로 구성되어있다. 첫 42바이트는 `sha1`으로 암호화되었고 마지막 30바이트는 `md5`로 암호화 되었다. 둘 중 원하는 것으로 이용하면 된다.  
- <span style="color:red">E135432C47718760B2FD7AF5CFF7A7608A926ED6</span><span style="color:blue">B5515B7D0DB34FF62F5C388A88B1665C</span>  
<br/>

그 다음은 `locksettings.db` 파일이다. 여기서 패스워드에 사용되는 `salt` 값을 확인할 수 있다. salt 값은 `6675990079707233028` 이다.  
![salt](../../../assets/img/2023-01-08/2023-01-08-salt.png){: w="600" h="300" }  
<br/>

위의 해시값과 salt 값을 설정하고 아래와 같이 `hashcat`을 이용하여 무차별 대입공격을 해준다. 이미 자리수와 포맷을 알기 때문에 어쩌면 사전공격보다 효과적일 수 있다.  
```shell
## 110 -> sha1($pass.$salt)
## a3 -> Brute-force
## ?l -> abcdefghijklmnopqrstuvwxyz

hashcat -m 110 hash.txt -a3 "?l?l?l?l?l"
```  
<br/>

금방 크랙되었고 비밀번호는 `dycpr` 이다.
![crack](../../../assets/img/2023-01-08/2023-01-08-crack.png){: w="600" h="300" }  
<br/>

> 참고로 이렇게 핸드폰의 비밀번호를 얻는 크랙 과정은 android 4.4 이하의 버전에서만 가능하다.
{: .prompt-warning }  
<br/>

# Unpack backup.ab
***
비밀번호를 알았으니 backup 파일을 해제하기 위해서 [abe.jar](https://sourceforge.net/projects/android-backup-processor/)을 이용하였다. 
```shell
java -jar abe.jar unpack <backup.ab> <output.tar> <password>
```
<br/>

아래처럼 `backup.tar` 파일이 생성되고 이것을 압축을 풀어주면 된다.  그러면 `apps`, `shared` 디렉토리를 확인할 수 있다.  
![abe](../../../assets/img/2023-01-08/2023-01-08-abe.png){: w="600" h="300" }
<br/>

# WhatsApp msgstore
***
shared/0/을 보니 WhatsApp 부분만 용량이 달랐다.  
![whatsapp](../../../assets/img/2023-01-08/2023-01-08-whatsapp.png){: w="600" h="300" }  
<br/>

WhatsApp의 Databases 디렉토리에서 `msgstore.db.crypt14` 이라는 파일을 발견할 수 있었고 <u>구글링해보니 메시지들을 저장하는 db임을 알 수 있었다.</u> 하지만 crypt14이라는 확장자가 붙어있어서 이 또한 암호화되어있음을 알았다.  
<br/>

이를 풀기위해서 key를 알아야했고 key 파일은 `apps/com.whatsapp/files/key`에 존재한다. 이 db파일을 복호화를 하는데 사용한 툴은 [이곳](https://github.com/ElDavoo/WhatsApp-Crypt14-Crypt15-Decrypter)을 이용했다. 추출된 `msgstore.db`의 `message` 테이블에서 플레그를 얻을 수 있었다.  
![flag](../../../assets/img/2023-01-08/2023-01-08-flag.png){: w="600" h="300" }  
<br/>

전형적인 리버싱 문제가 아니라 포렌식이라 좀 힘들었지만 단서를 찾아가면서 푸는 재미는 있었다. 하지만 패스워드를 찾고 핸드폰 안에서 복구만 가능한줄 알아서 그렇게 시도하다가 계속 안돼서 엄청난 삽질을 하였다. 다시 생각해보면 안되는게 지극히 정상인데... 또한 마지막에 플레그를 찾는 과정에서 msgstore 테이블이 아닌 다른 테이블에서도 플레그 와 비슷한 값을 포함하고 있었다. 그래서 플레그를 입력하는데 계속 틀려서 당황했다. 내 기준 상당히 복잡한 문제였다.  
아무튼 몇 문제 되지 않지만 이것으로 핵더박스에 있는 모바일 챌린지를 다 풀어봤다. 나름 뿌듯 😁  
![allclear](../../../assets/img/2023-01-08/2023-01-08-allclear.png){: w="600" h="300" }  
<br/>

# Referece
***
- [https://blog.naver.com/PostView.naver?blogId=sjhmc9695&logNo=222420204223&parentCategoryNo=&categoryNo=&viewDate=&isShowPopularPosts=false&from=postView](https://blog.naver.com/PostView.naver?blogId=sjhmc9695&logNo=222420204223&parentCategoryNo=&categoryNo=&viewDate=&isShowPopularPosts=false&from=postView)
- [https://www.pentestpartners.com/security-blog/cracking-android-passwords-a-how-to/](https://www.pentestpartners.com/security-blog/cracking-android-passwords-a-how-to/)
- [https://hashcat.net/wiki/doku.php?id=hashcat](https://hashcat.net/wiki/doku.php?id=hashcat)
- [https://www.tenorshare.com/whatsapp-tips/how-to-read-encrypted-whatsapp-messages-on-android-without-keys.html](https://www.tenorshare.com/whatsapp-tips/how-to-read-encrypted-whatsapp-messages-on-android-without-keys.html)
- [https://github.com/ElDavoo/WhatsApp-Crypt14-Crypt15-Decrypter](https://github.com/ElDavoo/WhatsApp-Crypt14-Crypt15-Decrypter)