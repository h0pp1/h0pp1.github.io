---
title: '[TIL] SSRF trick'
author: hoppi
date: 2022-12-24 22:14:00 + 0000
categories: [Web]
tags: [ssrf, php]
---
I found this interesting SSRF trick in a tweet.

This is a challenge for SSRF and the PHP code like this.  
The code checks `url` parameter whether scheme and host& path are correct or not, through `substr` function.  
![code](../../../assets/img/2022-12-24/2022-12-24-code.jpeg){: w="600" h="300" }
<br><br/>
As shown in below, we can not change red parts of url.  
We need to trigger SSRF by changing only 3 characters. So... How??

- <span style="color:red">https</span>://<span style="color:red">test.octagon.net/1.php</span>  
  
<br>  
As far as I know, some letters after the `@` character is treated as a hostname. And `0` means `localhost`.  
But I didn't expected like that.

- https<span style="color:GREEN">@0/</span>test.octagon.net/1.php/../../flag

<br>  
So I want to check out in my local.  
I just opened http server port 80 and made a request with weird payload.  
![curl](../../../assets/img/2022-12-24/2022-12-24-curl.png){: w="600" h="300" }  

<br> 
It worked! 🤣  
![result](../../../assets/img/2022-12-24/2022-12-24-result.png){: w="600" h="300" }  

# Reference
***
- https://twitter.com/octagonnetworks/status/1604915475753959438?s=46&t=1acsEgehBBspKpIEdNJavg