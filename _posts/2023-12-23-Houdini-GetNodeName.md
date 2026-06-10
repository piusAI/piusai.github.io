---
layout: post
published: true
title: "Houdini Get Node Name!"
subtitle: "GetNodename"
date: 2023-12-23 15:45:00 +0900
description: "Houdini String Attribute Controll"
categories: [Art]
tags: [Houdini, Art]
---

Vex와 Expression에서  
현재의 Node 이름을 가져오고 싶다!  
**$OS와 Vex에서 활용법**을 알아봅시다.


---

## Node Name in Expression, Vex

현재의 노드에서의 이름을, String으로 가져오기 위해서 어떻게 해야할까요?  
Expression에서의 활용과 Vex에서의 활용은 다르기 때문에 하나씩 알아보겠습니다.


### $OS, Nodename호출, in Expression
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN001.png" alt="GNN001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>sogi로 Assign된 Group</strong> </td> <td width="15%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN002.png" alt="GNN002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>

물론 Node 이름과 같이 sogi로 써서 해도 되지만 하나씩 작업하기 너무 귀찮습니다.

여러가지 Node가 있다면 하나씩 잡아야 하는 귀찮음을 감수해야겠죠?

  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN003.png" alt="GNN003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
$OS를 활용하여서 Group의 Name을 확인해보겠습니다.

  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN004.png" alt="GNN004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
똑같이 Group Name인 sogi로 point group이 적용되었습니다.


### ch와 $OS로 활용하기, in Vex
$OS는 Vex에서 활용키 어렵기 때문에,

편법아닌 편법으로 chs로 Channel function을 활용하여 string을 반환할 수있는 Expression을 열어주어  
거기에 $OS를 인식시키는 방법입니다.
  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN005.png" alt="GNN005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
  
``` hlsl
s@nodename=chs('nodename');
//nodename이라는 string attribute에 chs로 작성할 수있도록 칸을 열어줘
s@name=sprintf('sogi%dd',opdigits(@nodename));
//nodename에 있는 int정보를 sogi와 d사이에 넣어줘서 name이라는 string Attribute에 저장해줘.
```


[Houdini Vex Opdigit?](https://piusai.github.io/art/2023/12/21/Houdini-String-Opdigits.html)
opdigits에 관하여 궁금하시다면 위의 글을 참조해주세요.

  <table width="70%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN006.png" alt="GNN006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Assign된 Name Attribute</strong> </td>  </tr> </table>
  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN007.png" alt="GNN007" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
이렇게 많은 노드시에 유용 하겠죠?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/GetNode/GNN008.png" alt="GNN008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

이렇게 Name03 노드에서 확인할 수있는 spreadsheet입니다.