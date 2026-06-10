---
layout: post
published: true
title: Houdini Frame!
subtitle: GetNodename
date: 2024-02-19 15:45:00 +0900
description: Houdini String Attribute Controll
categories:
  - Art
tags:
  - Houdini
  - Art
---
Houdini에서 Frame을 가져와봅시다

Global 함수, Frame을 가지고 오고싶다!  
**$F, $FF**를 알아봅시다.

---

## Current Frame, 현재의 프레임을 가져오는 Global 함수
Global 함수로, 현재의 Frame을 가지고 올 수있습니다.

### $F vs $FF
#### $F

Int, 정수 타입의 Frame을 가져 오고 싶을 때 활용합니다.
Scatter Global Seed에 $F를 입력해보겠습니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Frame/F001.png" alt="F001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> <td width="100%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Frame/F002.png" alt="F002.png" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td> </tr> </table>
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Frame/F003.png" alt="F003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
**$F**는 Frame Number를 불러올 수 있지만, Substep을 켰을때도, int인 **정수타입의 Frame만 불러옵**니다.

#### $FF
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Frame/F004.png" alt="F004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/Frame/F005.png" alt="F005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong></strong> </td>  </tr> </table>
 이에 반해 **$FF**는 Current Frame을 **Float**, **substep까지 가져올 수있는 Global variable**입니다.