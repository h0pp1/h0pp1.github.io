---
title: '[Tip] Simple way to find the flag with re'
author: hoppi
date: 2023-04-07 23:08:00 + 0000
categories: [Tip]
tags: [re_module, python]
---

`re` in python is very useful module so it is used in a variety of ways. But in this tip, I'm just wanna talk about "`How to find the FLAG or string you want through simple way`". 
When trying wargame or CTF challenge, I'm always writing script or PoC code using python. Until relatively recently, I wrote it as below to find the flag.  
```python
...

## Get Flag
res = s.get(f'{URL}/admin')
FLAG = res.text.split('<p>')[1].split('here is ')[1].split('</p>')[0]
info(f'Flag is {FLAG}')
```
{: file="example.py"}
<br/>

It's ok when you get used to it, but it's not simple and clearly. And sometimes flag is hiding in sooo many duplicated tags. But as shown as below, there is simple way like this:  
```python
...

## Get Flag
res = s.get(f'{URL}/admin')
m = re.search(r"FLAG{.*?}", res.text)
if m:
    FLAG = m.group()
    info(f'Flag is {FLAG}')
else :
    fail("Failed to find the flag")
```
{: file="example.py"}
<br/>

Or when you need to extract string or key something in text, but What if key length is 128 or more?😩 No need to worry. You can use `regex` but also use `(?P<name>...)`. Then you just need to access `match.group("name")`. like this:
```python

res = sess.get(f"{URL}/key")
m = search(r"secret key is (?P<KEY>.*?) haha",res.text)
SOME_KEY = m.group("KEY") ## a0f3b49d7e20bc6a64c8e0c3d1537e5d5c5f6ca5c6e50948d2925d6dbcafc7e8d9c36d7b24fc84b01c7b8f92334ea712a86d0f613a42ec53fbf3c19d7205557

```
{: file="example2.py"}
<br/>
