---
layout: post
published: true
title: "Houdini String Attribute Get int : Opdigits"
subtitle: "What is Component in houdini"
date: 2023-12-21 15:45:00 +0900
description: "Houdini String Attribute Controll"
categories: [Art]
tags: [Houdini, Art]
---

Houdini에서 string에서 int를 가져와봅시다.

Vex   
문자열 정보에서 정수 정보를 가져오고 싶다!  
**opdigits**을 알아봅시다.

  

---

## String in Vex2, opdigits

string data안에 있는 int를 추출해보고 싶으면 어떤 vex를 활용해야 할까요?  

### String으로 만들기, opdigits  

예제를 먼저 만들어 볼게요

#### Voronoifracture 예제

  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO001.png" alt="SO001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Scatter 노드 Total Count - 4개</strong> </td>  </tr> </table>

voronoifracture를 활용하여 4개로 조각을 나누었습니다.

아래와 같이 이름을 Voronoifracture에서 수정을 할 수있지만,  
  
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO002.png" alt="SO002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="23%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO003.png" alt="SO003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
stringf를 활용하여 sogi3d로, 문자열로 반환 하여 봅시다.

[String vex란?](https://piusai.github.io/art/2023/12/19/Houdini-String-Vex.html)
(stringf를 모르시는 분들은 위의 게시글을 참조해주세요!)

또 추가적으로 vex를 활용할것이 있습니다!  
name 안에 있는 int을 문자열로 추출하여야 하겠습니다.  

#### opdigits, vex

string내에 있는 int, 정수를 반환할때 활용하는 opdigits!


  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO004.png" alt="SO004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Opdigits Vex</strong> </td>  </tr> </table>

[Opdigits Vex Houdini Docs](https://www.sidefx.com/docs/houdini/vex/functions/opdigits.html)

Output으로는 물론 int를 반환 시켜줍니다.


``` hlsl
s@name=sprintf('sogi%s%s',opdigits(@name),'d');
//@name에 있는 int를 sogi라는 string과 d라는 string 사이에 넣어서 함께 Output해줘.
```



  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO005.png" alt="SO005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>RunOver : primitives</strong> </td>  </tr> </table>



  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/StringOpdigits/SO006.png" alt="SO006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Primitives Attribute</strong> </td>  </tr> </table>

pieces name(string)에 있는 int를 뽑아서 다시 저장한 name attribute

  
