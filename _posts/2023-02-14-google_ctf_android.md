---
title: '[Mobile] GoogleCTF 2020 Android'
author: hoppi
date: 2023-02-14 18:21:00 + 0000
categories: [Mobile]
tags: [mobile, reversing, google_ctf]
math: true
---
좀 오래된 문제지만 GoogleCTF 2020에 출제되었던 모바일, 리버싱 문제를 풀어봤습니다.  
<br/>

# 분석
***
먼저 앱을 설치하여 화면 구성을 살펴보았습니다. 단순히 사용자의 입력을 받아서 `CHECK` 버튼을 누르게 되면 올바른지 아닌지 판단하는 것 같았습니다. 아래와 같이 아무 문자열을 입력했을 때 `X`표시를 띄워줍니다.  
![app](../../../assets/img/2023-02-14/app.png){: w="300" h="150" }  
<br/>

jadx를 통해서 앱을 까봤더니 제대로 디컴파일이 되지 않았습니다. 따라서 `onClick`에서 어떤 동작을 하는지 제대로 볼 수가 없었죠  
![jadx](../../../assets/img/2023-02-14/jadx.png){: w="700" h="350" }  
<br/>

특이한건 `R` 클래스 하단에 이런 `m1()` 이라는 메서드가 존재하는 것 외에는 별다른 것을 발견하지 못했습니다.  
![m1](../../../assets/img/2023-02-14/m1.png){: w="700" h="350" }  
<br/>

그다음으로 `ida`를 생각했는데 ida 같은 경우는 `dex` 파일을 던져주면 그것을 코드형태로 보여주지 못하는 것을 처음 알았습니다. 그래서 바로 `ghidra`에게 던져 주었습니다. 위 사진의 주석에 달린 것처럼 `renamed from ő`에 있는 라틴문자에 해당하는 클래스가 보였고 요 부분을 살펴보았습니다. 그리고 아래와 같이 `onClick`에 해당하는 부분을 볼 수 있었습니다. 그래서 속으로 `뭐지? 쉽네`라고 생각했지만 <u>이것은 페이크였습니다.</u>  
```java
void onClick(ő$1 this,View v)

{
  boolean bVar1;
  char cVar2;
  Integer pIVar3;
  Editable ref;
  String ref_00;
  String pSVar4;
  Object[] ppOVar5;
  StringBuilder ref_01;
  EditText ref_02;
  TextView pTVar6;
  int iVar7;
  Object ref_03;
  
  this.this$0.ő = 0;
  ppOVar5 = new Object[0x31];
  pIVar3 = Integer.valueOf(0x41);
  ppOVar5[0] = pIVar3;
  pIVar3 = Integer.valueOf(0x70);
  ppOVar5[1] = pIVar3;
  pIVar3 = Integer.valueOf(0x70);
  ppOVar5[2] = pIVar3;
  pIVar3 = Integer.valueOf(0x61);
  ppOVar5[3] = pIVar3;
  pIVar3 = Integer.valueOf(0x72);
  ppOVar5[4] = pIVar3;
  pIVar3 = Integer.valueOf(0x65);
  ppOVar5[5] = pIVar3;
  pIVar3 = Integer.valueOf(0x6e);
  ppOVar5[6] = pIVar3;
  pIVar3 = Integer.valueOf(0x74);
  ppOVar5[7] = pIVar3;
  pIVar3 = Integer.valueOf(0x6c);
  ppOVar5[8] = pIVar3;
  pIVar3 = Integer.valueOf(0x79);
  ppOVar5[9] = pIVar3;
  pIVar3 = Integer.valueOf(0x20);
  ppOVar5[10] = pIVar3;
  pIVar3 = Integer.valueOf(0x74);
  ppOVar5[0xb] = pIVar3;
  pIVar3 = Integer.valueOf(0x68);
  ppOVar5[0xc] = pIVar3;
  pIVar3 = Integer.valueOf(0x69);
  ppOVar5[0xd] = pIVar3;
  pIVar3 = Integer.valueOf(0x73);
  ppOVar5[0xe] = pIVar3;
  pIVar3 = Integer.valueOf(0x20);
  ppOVar5[0xf] = pIVar3;
  pIVar3 = Integer.valueOf(0x69);
  ppOVar5[0x10] = pIVar3;
  pIVar3 = Integer.valueOf(0x73);
  ppOVar5[0x11] = pIVar3;
  pIVar3 = Integer.valueOf(0x20);

...

  pIVar3 = Integer.valueOf(0x69);
  ppOVar5[0x2a] = pIVar3;
  pIVar3 = Integer.valueOf(0x6e);
  ppOVar5[0x2b] = pIVar3;
  pIVar3 = Integer.valueOf(0x67);
  ppOVar5[0x2c] = pIVar3;
  pIVar3 = Integer.valueOf(0x20);
  ppOVar5[0x2d] = pIVar3;
  pIVar3 = Integer.valueOf(0x6f);
  ppOVar5[0x2e] = pIVar3;
  pIVar3 = Integer.valueOf(0x6e);
  ppOVar5[0x2f] = pIVar3;
  pIVar3 = Integer.valueOf(0x3f);
  ppOVar5[0x30] = pIVar3;
  ref_01 = new StringBuilder();
  for (iVar7 = 0; iVar7 < ppOVar5.length; iVar7 = iVar7 + 1) {
    ref_03 = ppOVar5[iVar7];
    checkCast(ref_03,Character);
    cVar2 = ref_03.charValue();
    ref_01.append(cVar2);
  }
  ref_02 = this.val$editText;
  ref = ref_02.getText();
  ref_00 = ref.toString();
  pSVar4 = ref_01.toString();
  bVar1 = ref_00.equals(pSVar4);
  if (bVar1 == false) {
    pTVar6 = this.val$textView;
    pTVar6.setText("❌");
  }
  else {
    pTVar6 = this.val$textView;
    pTVar6.setText("🚩");
  }
  return;
}
```
<br/>

헥스값을 char로 치환해보면 `Apparently this is not the flag. What's going on?`임을 알 수 있습니다. 그래서 다른 부분을 살펴보던 중, 조금만 내려보면 `CatchHandlers`라는 문자열을 발견할 수 있었습니다.  
![catch](../../../assets/img/2023-02-14/catch.png){: w="700" h="350" }  
<br/>

그리고 무심코 클릭을 했는데 이런 오류가 뜨면서 메서드 부분이 디컴파일이 되지 않았습니다.  
> org.xml.sax.SAXParseException; lineNumber: 55; columnNumber: 15; Invalid byte 2 of 2-byte UTF-8 sequence.
{: .prompt-danger }  
<br/>

그래서 해당 오류를 구글링 했는데 [스택오버플로우](https://stackoverflow.com/questions/2421272/invalid-byte-2-of-2-byte-utf-8-sequence)에서 답을 찾을 수 있었습니다. 아마 라틴어로된 클래스이름, 파일이름 때문에 영향을 받는 것 같았습니다. 그래서 2가지 방법을 생각했었는데 첫 번째는 `AndroidManifest.xml` 파일에 인코딩 방식을 바꾸는 방법, 두 번째는 모든 `ő` 문자를 바꾸는 것이었습니다. 인코딩에 대해서 무지하기 때문에 후자를 택했는데 여기서 잠깐 롸업의 힘을 빌렸습니다. 
아래와 같이 커맨드를 입력하여 모든 `ő`를 `o`로 바꿔줍니다. 지금부터는 `o`라고 설명하겠습니다. (참고로 아래에서 gsed는 macOS에서 쓰는 sed 명령어 입니다)
1. `apktool d reverse.apk`
2. `find reverse/AndroidManifest.xml -type f | xargs gsed -i 's/ő/o/g'`
3. `find reverse/smali/ -type f | xargs gsed -i 's/ő/o/g'`
4. `find reverse/ -name "*ő*" -exec rename 's/ő/o/g' {} ";"`
5. `apktool b reverse -o replaced.apk`
<br/>

그리고 다시 추출한 `classes.dex`를 넣어주면 아래와 같이 코드를 볼 수 있습니다. 글 초반에 발견했던 `R.m1()`를 쓰는 것도 볼 수 있습니다. 일단 22 번째 줄에서 사용자의 입력값의 길이는 48입니다. 그리고 27 번째 줄에서 for문을 통해서 어떤 수를 생성하는데 입력값의 4글자씩 끊어서 모두 or 연산을 합니다. (여기서 비트 시프트가 들어가기 때문에 사실상 정수들을 더하기 한다고 보면 됩니다) 그다음 44 번째 줄에서 `R.o()` (renamed from R.m1())를 거치고 어떤 배열과 비교를 합니다.  
여기서 의문은 점은 로직상으로는 for문안에 어떤 배열과 비교하는 부분이 있어야하고 이중으로 돌아야할 것 같은데 비교부분이 블럭 밖에 존재합니다. 제 추측으로는 아래에 있는 `throwException()`을 통한 동작으로 강제적으로 루프를 돌던지 않을까 싶습니다... 아니면 플레그를 얻을 수가 없죠..?
```java
void UndefinedFunction_5003736a(void)

{
  long lVar1;
  char cVar2;
  Editable ref;
  String ref_00;
  int iVar3;
  int iVar4;
  int unaff_v1;
  EditText ref_01;
  TextView pTVar5;
  long[] plVar6;
  RuntimeException ref_02;
  o poVar7;
  
  ref_01 = unaff_v1.val$editText;
  ref = ref_01.getText();
  ref_00 = ref.toString();
  iVar3 = ref_00.length();
  if (iVar3 != 0x30) {
    pTVar5 = unaff_v1.val$textView;
    pTVar5.setText("❌");
    return;
  }
  for (iVar3 = 0; iVar4 = ref_00.length(), iVar3 < iVar4 / 4; iVar3 = iVar3 + 1) {
    plVar6 = unaff_v1.this$0.o;
    cVar2 = ref_00.charAt(iVar3 * 4 + 3);
    plVar6[iVar3] = (long)((int)cVar2 << 0x18);
    plVar6 = unaff_v1.this$0.o;
    lVar1 = plVar6[iVar3];
    cVar2 = ref_00.charAt(iVar3 * 4 + 2);
    plVar6[iVar3] = lVar1 | (int)cVar2 << 0x10;
    plVar6 = unaff_v1.this$0.o;
    lVar1 = plVar6[iVar3];
    cVar2 = ref_00.charAt(iVar3 * 4 + 1);
    plVar6[iVar3] = lVar1 & -0x100000000 | lVar1 & 0xffffffff | (long)((int)cVar2 << 8);
    plVar6 = unaff_v1.this$0.o;
    lVar1 = plVar6[iVar3];
    cVar2 = ref_00.charAt(iVar3 * 4);
    plVar6[iVar3] = lVar1 & -0x100000000 | lVar1 & 0xffffffff | (long)(int)cVar2;
  }
  plVar6 = R.o(unaff_v1.this$0.o[unaff_v1.this$0.o],0x100000000);
  if ((plVar6[0] % 0x100000000 + 0x100000000) % 0x100000000 !=
      unaff_v1.this$0.class[unaff_v1.this$0.o]) {
    pTVar5 = unaff_v1.val$textView;
    pTVar5.setText("❌");
    return;
  }
  poVar7 = unaff_v1.this$0;
  poVar7.o = poVar7.o + 1;
  if (unaff_v1.this$0.o.length <= unaff_v1.this$0.o) {
    pTVar5 = unaff_v1.val$textView;
    pTVar5.setText("🚩");
    return;
  }
  ref_02 = new RuntimeException();
  throwException(ref_02);
  return;
}
```
<br/>

일단 비교하는 배열은 `o.java`의 생성자 부분에서 발견할 수 있었습니다. 옆으로 쭉 가보면 길이가 12였기 때문에 우리가 원하는 배열임을 추측할 수 있죠. (정확하게는 smali 코드안에서도 확인하는게 좋을 듯 합니다)  
![data](../../../assets/img/2023-02-14/data.png){: w="700" h="350" }  
<br/>

# Get Flag
***
아래와 같이 `Brute-force`하는 코드를 작성할 수 있습니다. 플레그의 포맷을 정확히 모르기 때문에 `string.printable`를 이용하였는데 이게 100개이기 때문에 4글자를 돌릴려면 최악의 경우 1억번 확인을 해야합니다🥲 그 전에 나오긴 하겠지만 12글자를 확인해야하니 좀 오래걸릴 것 같습니다.  
<br/>

## Exploit Code
***
```python
from string import printable
from time import sleep
info = lambda x: print(f'[+] {x}')

data = [40999019, 2789358025, 656272715, 18374979, 3237618335, 1762529471, 685548119, 382114257, 1436905469, 2126016673, 3318315423, 797150821]
FLAG = ''
FIND = 0

def o(a, b) :
   if (a == 0) :
       return [0, 1]

   r = o(b % a, a)
   return [(r[1] - (((b // a) * r[0]))), r[0]]

if __name__ == '__main__':
    printable_strs = printable
    printable_length = len(printable_strs)
    for i in range(len(data)):
        for str1 in printable_strs:
            if FIND == 1:
                break
            for str2 in printable_strs:
                if FIND == 1:
                    break
                for str3 in printable_strs:
                    if FIND == 1:
                        break
                    for str4 in printable_strs:
                        if FIND == 1:
                            break
                        value = ord(str1) + (ord(str2) << 8) + (ord(str3) << 16) + (ord(str4) << 24)
                        value = o(value,0x100000000)
                        if((value[0] % 0x100000000 + 0x100000000) % 0x100000000) == data[i]:
                            FLAG += str1+str2+str3+str4
                            info(f'{FLAG}')
                            sleep(3)
                            FIND = 1
                            break
        FIND = 0
    info(f'Flag is {FLAG}')
```
{: file='exploit.py'}
<br/>

다행이 아래와 같이 플레그를 확인할 수 있었습니다.  
![flag](../../../assets/img/2023-02-14/flag.png){: w="700" h="350" }  
<br/>

![flag2](../../../assets/img/2023-02-14/flag2.png){: w="300" h="150" }  
<br/>

디컴파일이 제대로 동작하지 않는 첫 문제였는데 `ghidra`도 자주 이용해야 할 것 같고 리버싱을 할 때 가끔은 믿음이 필요하다는 것을 느낀 문제였습니다🙃
<br/>

# Reference
***
- [https://github.com/luker983/google-ctf-2020/tree/master/reversing/android](https://github.com/luker983/google-ctf-2020/tree/master/reversing/android)
- [https://stackoverflow.com/a/1585189](https://stackoverflow.com/a/1585189)
- [https://stackoverflow.com/a/1585189](https://stackoverflow.com/a/1585189)
- [https://stackoverflow.com/a/9394874](https://stackoverflow.com/a/9394874)