---
title: '[Hackthebox] Encryption Bot'
author: hoppi
date: 2023-03-09 18:30:00 + 0000
categories: [Reversing]
tags: [reversing]
math: true
---

리버싱 분야에서 Easy 난이도로 새로운 문제가 나왔길래 한번 풀어보았습니다. 하지만 리버싱 초보인 저에게는 그렇게 쉽지는 않았다는게 함정...   
<br/>

**키워드: Reversing**  

![description](../../../assets/img/2023-03-09/description.png){: w="700" h="350" }  
<br/>

# 분석
***
ida를 통해 `main`함수를 보면 아래와 같습니다. 분석해야할 함수는 `55DC`, `531D`, `54BA` 총 3가지입니다. (중간중간 아무런 동작하지 않는 함수는 fake로 변경하였습니다)  
```c
__int64 __fastcall main(__int64 a1, char **a2, char **a3)
{
  char v4[32]; // [rsp+0h] [rbp-30h] BYREF
  int v6; // [rsp+2Ch] [rbp-4h]

  sub_5628();                          // title
  v6 = 0;
  printf("\n\nEnter the text to encrypt : ");
  __isoc99_scanf("%s", v4);
  if ( fopen("data.dat", "r") )
    system("rm data.dat");
  putchar(10);                                  // line feed
  sub_55DC(v4);                         // check input length
  fake3();
  sub_531D(v4);
  sub_54BA();
  system("rm data.dat");
  putchar(10);                                  // line feed
  return 0LL;
}
```
<br/>

먼저 `55DC()`는 단순히 사용자의 입력 값이 27자 인지 아닌지를 판단하고 있습니다. 프로그램이 종료되지 않기 위해서는 27자를 입력해야겠죠.  
![55dc](../../../assets/img/2023-03-09/55dc.png){: w="700" h="350" }  
<br/>

다음 `531D()`의 경우 사용자의 입력 값을 v3 배열에 한 글자씩 저장한 뒤 이것을 `51D9()`의 인자 값으로 넣어 주고 있습니다.  
![531d](../../../assets/img/2023-03-09/531d.png){: w="700" h="350" }  
<br/>

`51D9()`으로 가보면 한 글자씩 비트형식으로 변환 후 파일 스트림에 씁니다. 그렇다면 <u>27자 * 8비트 이므로 0과 1로 구성된 216 길이의 데이터</u>가 `data.dat` 파일에 저장되어 있을 겁니다.  
![51d9](../../../assets/img/2023-03-09/51d9.png){: w="700" h="350" }  
<br/>

`main`의 마지막 메서드인 `54BA()`를 보면 다음과 같습니다. 파일에서 0 또는 1 한 글자씩 가져와서 v1 배열에 넣어주고 i가 6이되면, 즉 비트가 6개가 될 때 `53AB()`와 `53E9()`를 이용합니다.  
```c
int sub_5555555554BA()
{
  int v1[350]; // [rsp+0h] [rbp-660h]
  int v2; // [rsp+640h] [rbp-20h]
  char v3; // [rsp+647h] [rbp-19h]
  FILE *stream; // [rsp+648h] [rbp-18h]
  int j; // [rsp+650h] [rbp-10h]
  int v6; // [rsp+654h] [rbp-Ch]
  int v7; // [rsp+658h] [rbp-8h]
  int i; // [rsp+65Ch] [rbp-4h]

  stream = fopen("data.dat", "r+");             // 파일이 존재하면 읽고 쓰기가 가능. 없으면 에러
  fake();
  for ( i = 1; i <= 216; ++i )
  {
    v3 = fgetc(stream);
    if ( v3 == 48 )
    {
      v1[i - 1] = 0;
    }
    else if ( v3 == 49 )
    {
      v1[i - 1] = 1;
    }
    if ( i && !(i % 6) )
    {
      v7 = i - 1;
      v6 = 0;
      for ( j = 0; j <= 5; ++j )
      {
        v2 = sub_5555555553AB(j);
        v6 += v2 * v1[v7--];
      }
      sub_5555555553E9(v6);
    }
  }
  return fclose(stream);
}
```

`53AB()`는 n을 인자로 받아 $2^n$을 반환합니다. 즉, 위 코드 `29` 번째 줄에 있는 for문은 <u>6비트씩 다시 숫자로 바꾸는 과정</u>임을 알 수 있습니다.  
![53ab](../../../assets/img/2023-03-09/53ab.png){: w="700" h="350" }  
<br/>

`53E9()`는 인자로 넘겨받은 인덱스에 해당하는 문자를 암호문 문자열에서 찾아 반환합니다.  
![53e9](../../../assets/img/2023-03-09/53e9.png){: w="700" h="350" }  
<br/>

암호화 동작과정을 정리하면 아래와 같습니다.  
1. 27자 길이의 문자를 입력합니다.
2. 입력받은 문자를 하나하나 바이너리 형태로 바꿉니다. ($$27 \times8=216$$)
3. 이 바이너리를 다시 6비트씩 자릅니다. ($$216 \div6=36$$)
4. 6비트씩 자른 것을 다시 숫자형태로 치환합니다. (6비트이니 0~63까지)
5. 문자열에서 이 숫자를 인덱스로 삼아서 해당 위치의 문자를 반환합니다. 이 과정을 36번 반복하여 길이가 36인 암호화된 문자열이 생성됩니다.
<br/>

그림으로 쉽게 나타내면 다음과 같겠죠? 이해가 된다고 해주세요🥲  
![pic](../../../assets/img/2023-03-09/pic.jpeg){: w="700" h="350" }  
<br/>

# Get Flag
***
암호화된 문자열을 알기 때문에 이것을 암호문 문자열에 대응시켜 해당하는 인덱스를 얻어온 뒤, 이 인덱스(숫자)들을 바이너리화하고 비트를 8개씩 묶어서 다시 숫자로 변환하여 플레그를 얻을 수 있습니다. 암호화하는 코드, 복호화하는 코드 둘 다 작성해봤습니다.  
<br/>

## Encryption Code
***
```python


v3 = [0 for i in range(216)]

def FIVE31D(input):
    for i in range(27):
        print(input[i])
        FIVE1D9(ord(input[i]), i*8)
        print()

def FIVE1D9(input_int, index):
    
    for i in range(7,0,-1):
        v3[i+index] = input_int % 2
        input_int= input_int // 2
    for i in range(8):
        print(v3[i+index],end="")

def FIVE4BA():
    v1 = [0 for i in range(350)]
    for i in range(216):
        if v3[i] == 0:
             v1[i+1-1]=0
        elif v3[i] == 1:
             v1[i+1-1]=1
        if ((i+1) and not((i+1)%6)):
            v7 = i+1-1
            v6 = 0
            for j in range(6):
                v2 = FIVE53AB(j)
                v6 += v2 * v1[v7]
                v7 -= 1
            FIVE3E9(v6)

def FIVE53AB(j):
    v3 = 1
    for i in range(j):
        v3 *=2
    
    return v3

def FIVE3E9(v6):
    string = "RSTUVWXYZ0123456789ABCDEFGHIJKLMNOPQabcdefghijklmnopqrstuvwxyz"
    print(string[v6],end="")
    # print(string[v6],end="")


if __name__ == '__main__':
    flag = "HTB{I_4M_R3v3rse_EnG1n3eR!}"
    FIVE31D(flag)
    print(v3)
    FIVE4BA()
```
{: file="Encrypt.py"}
<br/>

## Decryption Code
***
```python
info = lambda x :print(f'[+] {x}')


def flag_enc_to_index():
    for i in range(36):
        for j,char in enumerate(string):
            if flag_enc[i] == string[j]:
                reverse_flag_int[i] = j

def index_to_binary(num, index):
    for i in reversed(range(6)):
        bits[i+index] = num % 2
        num= num // 2
    # for i in range(6):
    #     print(bits[i+index],end="")

def binary_to_str(index):
    sum = 0
    for j in reversed(range(8)):
        sum += bits[j+index] * REVERSE_FIVE53AB(j)
    flag_int[index//8] = sum        

def REVERSE_FIVE53AB(j):
    num = 129
    for i in range(j):
        num //=2

    return num 




if __name__ == '__main__':
    bits = [0 for i in range(216)]
    flag_enc = "9W8TLp4k7t0vJW7n3VvMCpWq9WzT3C8pZ9Wz"
    string = "RSTUVWXYZ0123456789ABCDEFGHIJKLMNOPQabcdefghijklmnopqrstuvwxyz"
    reverse_flag_int = [0 for i in range(36)]
    flag_int = [0 for i in range(27)]
    FLAG = ''

    flag_enc_to_index()
    # print(reverse_flag_int)

    for i in range(36):
        index_to_binary(reverse_flag_int[i], i*6)
        # print()
    

    for i in range(27):
        binary_to_str(i*8)
    
    # print(flag_int)
    
    for i in range(27):
        FLAG += chr(flag_int[i])
    
    
    info(f'Flag is {FLAG}')
```
{: file="decrypt.py"}
<br/>

새벽에 풀려고 해서 그런지 엄청나게 쉬운 문제는 아니라고 생각했는데 난이도 투표를 보니 초록초록해서 마음이 아픕니다...  

