---
title: '[TIL] Overflow PHP output buffer'
author: hoppi
date: 2023-03-06 00:55:00 + 0000
categories: [Web]
tags: [web, php_output_buffer, xss]
published: false
---

제가 즐겨보는 트윗인 `INTIGRITI`의 `spot the vulnerability` 챌린지에서 본 재밌는 내용을 적어보려고 합니다. 일단 code snippet은 아래와 같습니다.  
```php
<?php
if (isset($_GET['email']))
  $email = filter_var($_GET['email'],
                      FILTER_SANITIZE_EMAIL);
if (isset($_GET['xss']))
  $xss = htmlspecialchars($_GET['xss']);
if (isset($_GET['path'])) {
  $path = $_GET['path'];
  while (strpos($path, '../') !== false) {
    $path = str_replace('../', '', $path);
    if (isset($_GET['debug'])) {
      echo '[DEBUG] Removed \'../\'. New path is ';
      echo htmlspecialchars($path);
}}} ?>
<?php
header("content-security-policy:default-src 'none'");
?>
<h1>Sanitization as a Service</h1>
<p>We're revolutionizing the world of sanitization!
<br>Just submit the string you want sanitized,
and we'll do all the hard work!</p>
<h6>Here's your sanitized string:</h6>
<p>Email: <?php echo $email; ?></p>
<p>Xss: <?php echo $xss; ?></p>
<p>Path: <?php echo $path; ?></p>
```
{: file='index.php'}
<br>

위 코드를 보면 GET 파라미터로 사용자의 입력 값을 전달할 수 있죠. 6 번째 줄에서 전달받은 `xss` 파라미터의 값을 `htmlspecialchars()`를 통해서 `XSS`에 쓰이는 문자를 거르고 있습니다. 하지만 13 번째 줄을 보면 `$path`도 `sanitize`를 수행하지만 다른 변수에 값을 저장하지 않고 단순히 `echo`만 합니다. 따라서 코드 마지막 줄에도 그대로 들어가기 때문에 `XSS`가 될 것처럼 보입니다. 하지만 코드를 쭉 보신분들을 아시겠지만 16 번째 줄에서 `CSP`를 통해 `default-src` 지시문을 `none`으로 설정하고 있습니다. 아무것도 허용하지 않겠다는 뜻이죠. 그럼 `XSS`가 트리거되지 않는게 정상이지만 아래 사진처럼 이를 우회하고 트리거할 수 있습니다. 어떻게 가능할까요?🤔  
![bypass](../../../assets/img/2023-03-06/bypass.png){: w="600" h="300" }  
<br/>


PHP 에서는 `output buffer`라는 것을 사용합니다. 사용자가 지정하지 않으면 디폴트 값을 4096 바이트로 설정되어있습니다. PHP 공식문서에서 `output_buffering`에 대한 설명은 아래와 같습니다.  
> output_buffering bool/int : You can enable output buffering for all files by setting this directive to 'On'. If you wish to limit the size of the buffer to a certain size - you can use a maximum number of bytes instead of 'On', as a value for this directive (e.g., output_buffering=4096). This directive is always Off in PHP-CLI.
{: .prompt-info }
<br/>

먄약 이것을 넘는 데이터를 주게되면 위 코드가 어떻게 동작할까요? 아래처럼 경고 메시지를 받을 수 있습니다. 읽어보면 이미 헤더에 대한 정보를 보냈다고 합니다. <u>따라서 16 번째 줄에서 설정한 CSP 헤더는 설정되지 않는 것이죠 ㄷㄷ.</u>  
![warning](../../../assets/img/2023-03-06/warning.png){: w="700" h="350" }  
<br/>

코드를 보면서 눈치 빠르신 분들은 `debug 파라미터가 괜히 있지는 않을거같은데...` 라는 생각을 하셨을겁니다. while 문을 돌면서 `../`문자열이 없어질 때까지 `[DEBUG] Removed '../'. New path is`와 우리가 입력한 `sanitize`된 `$path`를 반복적으로 보여주고 있습니다. 따라서 `$path`에 적절한 값만 주면 `output buffer`가 금방찰 수 있게됩니다. 아래와 같은 페이로드를 이용할 수 있습니다.  
```plaintext
................................../////////////////""""""""""""""""""""""""""<svg/onload=alert`hahahoho`>
```
{: file='payload'}
<br/>

`alert()`을 수행하고 난 모습입니다. console에도 `CSP`에 대한 경고 없이 깔끔한 모습을 볼 수 있습니다🙃
![trigger](../../../assets/img/2023-03-06/trigger.png){: w="700" h="350" }  
<br/>


# Reference
***
- [https://twitter.com/intigriti/status/1628736484365795329?s=61&t=DIT-LhtH3WG0MFLbmZlg4A](https://twitter.com/intigriti/status/1628736484365795329?s=61&t=DIT-LhtH3WG0MFLbmZlg4A)
- [https://stnv.medium.com/intigritis-march-xss-challenge-by-brunomodificato-3740a0c05948](https://stnv.medium.com/intigritis-march-xss-challenge-by-brunomodificato-3740a0c05948) 