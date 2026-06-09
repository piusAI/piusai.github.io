---
layout: post
published: true
title: "Houdini String Vex?"
subtitle: "What is Component in houdini"
date: 2023-12-19 15:45:00 +0900
description: "Houdini String 붙히기"
categories: [Art]
tags: [Houdini, Art]
---

Houdini Vex에서 string을 활용해봅시다.

Vex   
문자열을 string으로 가져오고 싶다!  
**Sprintf, concat, **itoa****을 알아봅시다.  
(feat. padzero expression)

---

## String in Vex
Name attribute set시, Primitive에서 가장 중요할 수도 있는 String, 문자열을 Vex에서의 활용을 알아볼까요?  
concat, itoa, sprintf, padzero(expression)를 알아볼 것입니다.


### Concat, itoa

#### concat, vex
Concat은 문자열을 형성하는 Vex function입니다.  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS000.png" alt="VS000" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Concat Vex</strong> </td>  </tr> </table>

용도는, Concat으로 String을 연결할 수 있습니다.  
OutPut으로는 String을 반환을 하겠네요.
``` hlsl
s@name= concat('sogi','3d');
//sogi라는 string과 3d라는 string을 연결해서 name이라는 string Attribute에 넣어줘.
```


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS001.png" alt="VS001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Detail Vex</strong> </td> <td width="25%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS002.png" alt="HC012" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Detail Attribute</strong> </td> </tr> </table>
: Detail Attribute에 들어간 name

그렇지만, String만을 연결하려면 아래와 같이 그냥 s@name= 'sogi3d'만을 활용해도 되겠죠?
``` hlsl
s@name='sogi3d';
//sogi3d라는 string을 name이라는 string Attribute에 넣어줘.
```

그렇기 때문에 우리는 integer와 함께 더하여 string에 저장할 수 있도록 itoa를 접목시켜볼까요?

#### itoa, vex
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS003.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>itoa Vex</strong> </td>  </tr> </table>
되게 간단합니다. Integer, 즉 정수를 string, 문자열로 반환하는 Vex네요.  
그렇다면 Concat과 함께 써볼까요?

``` hlsl
s@name= concat('sogi', itoa(@Frame), 'd');

//"sogi"와 "Frame번호"와 "d"를 붙혀서 string으로 name에 attribute에 저장해줘.
```


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS004.png" alt="VS004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Frame 번호에서부터</strong> </td> <td width="58%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS005.png" alt="VS005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Integer값으로 간다</strong> </td> </tr> </table>


### Sprintf, padzero

#### sprintf, vex
sprintf vex도 물론 문자열을 반환하는 vex입니다.

  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS006.png" alt="VS006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Sprintf Vex</strong> </td>  </tr> </table>

``` hlsl
s@name=sprintf('sogi%03dd',@Frame);
//"sogi"와 "Frame번호 세자리수"와 "d"를 붙혀서 string으로 name에 attribute에 저장해줘.

s@name=sprintf('sogi%01d%s',@Frame,'d');
//"sogi"와 "Frame번호 한자리수"와 "d"를 붙혀서 string으로 name에 attribute에 저장해줘.
```

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS007.png" alt="VS007" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="55%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS008.png" alt="VS008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>

#### padzero, Expression


  <table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS009.png" alt="VS009" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Padzero Vex</strong> </td>  </tr> </table>

주어진 길이만큼 숫자를 0으로 채운 문자열 반환하는 padzero  
stringf와는 또 다른 활용도가 있겠습니다.

주의!  
위의 padzero는 expression이기 때문에 vex처럼 wrangle에서 활용할 수없습니다.  

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS010.png" alt="VS010" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/VexString/VS011.png" alt="VS011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>

MMB로 절대 경로 확인

padzero를 활용하여서 Filecache에 활용한 모-오습.  

``` hlsl
$HIP/sogi`padzero(3,chs("sogi"))`d/$OS.$F04.bgeo.sc

//C:/Houdini/sogi003d/padzero.0003.bgeo.sc
//padzero의 의미 : Channel sogi를 가져와서 세자리수로 불러와줘

$HIP/sogi`padzero(1,chs("sogi"))`d/$OS.$F04.bgeo.sc
//C:/Houdini/sogi3d/padzero.0003.bgeo.sc
//padzero의 의미 : Channel sogi를 가져와서 한자리수로 불러와줘
```

