---
layout: post
published: true
title: "Houdini Turbulence Noise?"
subtitle: "What is Tubulence"
date: 2023-12-15 15:45:00 +0900
description: "Houdini Level"
categories: [Art]
tags: [Houdini, Art]
---
Houdini Vop이란?
VOP : Noise때문에 Vop을 쓴다고해도 과언이 아닐 텐데요.  
**Turbulunce Noise**에 대하여 알아봅시다.

---

## VOP, Turbulent

Vop에서의 가장 강점중의 하나는 **Noise의 활용** 입니다.  
Noise를 활용하여 사실적인 Simulation을 만드는데 가장 이용도가 높을 것 인데요, Map에 Pattern이 다른 Noise Type중에서, 가장 활용도가 높은 Turbulent Noise에 관하여 이야기 해볼까요?

  

### Turbulent Noise Setting

Grid의 Color에다가 Noise를 잡아 주겠습니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN001.png" alt="TN001" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Color Setup을 활용하기 위한, 기본 Sop Settings]</strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN002.png" alt="TN002" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[기본 set-up]</strong> </td>  </tr> </table>

#### 01 Signature

가장 첫번째로 눈에 들어오는 것은 Signature입니다.  
Output으로 1DNoise인지, 3D Noise인지에 따라서 다르겠죠?

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN003.png" alt="TN003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>[Signature : 3D Noise]</strong> </td>  </tr> </table>
1D Noise는 Float값을 Output하며, R=G=B인 Gray Scale(GR)의 Color를 변환시켜준 이후


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN004.png" alt="TN004" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Color rgb에 들어간 모습</strong> </td>  </tr> </table>

3D noise는 Vector 값을 Output하기에, R≠G≠B인 RGB의 Color를 변환시켜줍니다.

Vector값을 정확히 넣어주어야 점선인 Noise > Cd연결선이 실선으로 변화는것을 확인할 수있네요!

Vector(Rgb)를 써야한다, Float(GR)를 써야한다에서는 정답이 없지만, 필요시 RGB에도 Random을 넣을 수 있어야겠죠?


#### 02 Noise Type

두번째는 Noise Type 이겠죠?


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN005.png" alt="TN005" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Noise Type을 활용하기 위한, 기본 Sop Settings</strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN006.png" alt="TN006" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> TurbNoise 1D</strong> </td>  </tr> </table>

기본 Setup을 하기 위해 확인을 해보니, 이렇게 점으로 Output 됩니다.

그 이유는 @P를 받은 pos에서 현재 Turbnoise에서 연산과정을 거쳐 P로 들어가서 Output이 되겠죠?

이 Setup이 틀린 것 만은 아닙니다.  
우리가 Noise Type의 Output을 확인하기 위해서 이렇게 활용 해보면 가장 정확할 것 같습니다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN007.png" alt="TN007" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Alligator Noise Output 0~1 </strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN008.png" alt="TN008" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Perlin Noise Output 0.5~1.5 </strong> </td>  </tr> </table>

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN009.png" alt="TN009" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Original Perlin Noise Output -0.5~0.5 </strong> </td>  </tr> </table>



#### 03 Frequency | Offset | Amp | Roughness | Attenuation | Turbulence

마지막 세팅값

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN010.png" alt="TN010" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> P.y에만 Noise 더해주기위한 기본 set up </strong> </td>  </tr> </table>

Vector to float으로 X와 Z는 Float to vector로 받아 다시 P로 보내주고

Vector to float으로 Y는 Noise와 더해져서 @P.y로 보내줍니다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN011.png" alt="TN011" style="width: 100%; max-width: 100%; height: auto;"> <br><strong> Noise TypeType : Original Perlin Noise </strong> </td>  </tr> </table>

음양수 P.y를 Noise로 주기 위하여 Original Perlin Noise를 활용함.

  

##### ㄱ. Frequency(빈도)

높은 값일수록 Noise의 크기가 더 작아지고  
낮은 값일수록 Noise의 크기가 더 커진다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN012.png" alt="TN012" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Frequency(2, 2, 2)</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN013.png" alt="TN013" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Frequency(0.5, 0.5,0.5)</strong> </td> </tr> </table>

    
##### ㄴ. Offset

xyz축으로 Offset함
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN014.gif" alt="TN014" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Offset X</strong> </td>  </tr> </table>
x축으로 Offset하는 예시


##### ㄷ. Amplitude, 진폭

Noise의 최소값와 최대값 크기
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN015.png" alt="TN015" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Amplitude : 2</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN016.png" alt="TN016" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Amplitude : 0.5</strong> </td> </tr> </table>

     

##### ㄹ. Roughness, 거칠기

Fractal Noise의 Iteration Scale Increment 추가
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN017.png" alt="TN017" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Roughness : 0.15</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN018.png" alt="TN018" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Roughness : 0.8</strong> </td> </tr> </table>


##### ㅁ. Attenuation, 감쇠

극심한 스파이크 방지를 위해 Noise 평탄화  
값이 높을수록 더 부드럽고, 낮을 수록 더 차이가 심해진다.

<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN019.png" alt="TN019" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Attenuation : 0.4</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN020.png" alt="TN020" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Attenuation : 1.6</strong> </td> </tr> </table>


##### ㅂ. Turbulence, 난기류

Noise의 정도 제어,
값이 높을수록 더 많은 봉우리로 인해 더 Chaostic하게 만들고, 덜 부드럽다  
값이 낮을수록 적은 봉우리, 더 부드럽다.
<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN021.png" alt="TN021" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Turbulence : 0</strong> </td> <td width="50%" style="text-align: center; border: none; padding: 5px;"> <img src="/assets/postimg/Houdini/TurbulenceNoise/TN022.png" alt="TN022" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Turbulence : 10</strong> </td> </tr> </table>