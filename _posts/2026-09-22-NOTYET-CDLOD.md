---
layout: post
published: false
title: Concurrent Binary Trees
thumbnail-img: /assets/img/Renderpipeline.jpg
date:   2026-09-21 18:32:00 +0900
description: Paper Keyword?
categories:
  - Graphics
tags:
  - Graphics
author: PIUS
---

## CDLOD
CDLOD : Continuous Distance - Dependent LOD



[CDLOD paper](https://aggrobird.com/files/cdlod_latest.pdf)


<table width="100%" style="table-layout: fixed; border-collapse: collapse; border: none;">
  <tr style="border: none;">
    <td width="100%" style="text-align: center; border: none; padding: 5px;">
      <a href="https://youtu.be/lNwVZfe7WL8" target="_blank" style="position: relative; display: inline-block;">
        <img src="https://img.youtube.com/vi/lNwVZfe7WL8/maxresdefault.jpg" alt="영상 미리보기" style="width: 100%; max-width: 100%; height: auto;">
        <span style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 68px; height: 48px; background: url('https://upload.wikimedia.org/wikipedia/commons/b/b8/YouTube_play_button_icon_%282013%E2%80%932017%29.svg') no-repeat center/contain;"></span>
      </a>
      <strong>UE Water Plugin</strong>
    </td>
  </tr>
</table>
- [Water MeshComponent](https://github.com/EpicGames/UnrealEngine/blob/release/Engine/Plugins/Experimental/Water/Source/Runtime/Public/WaterMeshComponent.h)를 활용하면 위와 같은 CDLOD를 활용 할 수있었다


* **장점**
  - T-Junction과 같은 Crack이 발생하지 않는다
  - Morphing / Interploate로 vtx들이 자연스럽게 Conversion이 일어남
* **단점**
  균일한 
  
Lod Level간의 transition이 매끄럽다 pop-artifact가 없다
-> 이를 Stitiching Strip을 사용함으로 해결
SM3이상 지원하는 Graphics HW에서 작동!

## Algorithm implementation
##### Concepts

- Quadtree dpeth Level이 항상 LOD level에 대응

> " child node has Four Times More Mesh Complexity per square area unit than its parent "  

→ Quad Tree로 부모에서 자식으로 내려갈때 1/4영역 담당


> "approximately the same average number of triangles per square unit of screen over the whole rendered terrain"

Perpective projection에서 Camera - Object 2배 멀어진다 : 단위면적당 1/2<sup>2</sup>  
거리 `2배` : 화면 면적 `1/4`, LOD 레벨 하나 coarse -> 밀도 `1/4`로 상쇄

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CDLOD/CDLOD_Morph.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Morph range</strong> </td>  </tr> </table>
>"To provide Smooth LOD Transition, the Morph area is also defined,  ...
>This Morph area usually covers the Last 15% ~ 30% of every LOD range.

→각 LOD range에서 15% ~ 30%구간 : `Morph area`로, Lod Smooth LOD 변형이 일어남

##### 순회 방향
> "Traversed recursievely begining from the most distant nodes of the lowest - detailed level, working down to the closet, most detailed ones"


##### QuadTree traversal -pseudo Code
``` cpp

// LOD levl은 최대 레벨에서 시작해 0까지 내린다. LOD0이 가장 섬세한 LOD
bool Node::LODSelect(int ranges[], int LodLevel, Frustum frustum)
{
    //이 LOD범위 밖 : 너무 멀다. 부모( 더 Coarse한 LOD)가 영역을 그려야함
    if(!nodeBoundingBox.IntersectsSphere(ranges[LodLevel]))
        return false;
    
    // 범위 안이지만 Frustum밖 : 아무것도 선택하지 않고 "True(처리완료)"표시
    // false를 주면 부모가 이 영역을 더 거친(coarse) LOD로 그린다?
    if(!FrustumIntersect(frustum))
        return true;
        //어차피 아무것도 안그림(culling)
        
    // 가장 섬세한 레벨 : 더 내려갈 곳 없으니 노드 전체 선택..?
    if( LodLevel ==0)
    {
        AddWholeNodeToSelectionList();
        return true;
    }
    
    //LOD0이 아니니까 더 detail한 LOD level을 위해 child로 내린다
    else
    {
        // 더 상세한 레벨(LodLevel-1)의 범위에는 안 들어감
        // -> 현재 Level이 적당하므로 노드 전체 선택..!
        if(!nodeBoundingBox.IntersectsSphere(ranges[LodLevel-1]))
        {
            //필요한 LODLevel범위 커버
            AddWholeNodeToSelectionList();
        }
        else
        {
            //더 상세한 LOD Level범위 커버 -> 자식들에게 위임
            foreach(childNode)
            {
                if(!childNode.LODSelect(ranges, LodLevel-1, frustum))
                {
                    // 자식이 범위 밖이라 처리 못함
                    // -> 그 자식 영역(사분면)만 현재 노드 메시로 대신 그림
                AddPartOfNodeToSelectionList(childNode.ParentSubArea);
                }
            }
        }
        return true; // 이 노드 영역은 처리 완료
    }
}

...
void Node::AddPartOfNodeToSelectionList( SubArea area)
{
    SelectionList.push_back({this, area});
}
```


| Return | 상황                                            | 동작                             |
| ------ | --------------------------------------------- | ------------------------------ |
| true   | 범위 안 + **프러스텀 밖**                             | 아무것도 선택 안 함 (컬링)               |
| true   | 범위 안 + **LodLevel == 0**                      | 노드 전체 선택                       |
| true   | 범위 안 + **현재 레벨이 적당** (`ranges[LodLevel-1]` 밖) | 노드 전체 선택                       |
| true   | 범위 안 + **더 상세 필요** (`ranges[LodLevel-1]` 안)   | 자식에게 위임, 범위 밖 자식 사분면만 내 메시로 채움 |
| false  | `ranges[LodLevel-1]`안                         | 부모가 그려라!                       |
둘다 처리는 끝난것이고, `true`는 처리 끝남, `false`도 처리는 끝났지만, 결과가 부모로 위임!  


- LOD는 Main camera기준, ShadowMap은 다른 Camera 기준

#### vtx Displacement가 Patch를 만드는 방식

> "a single grid mesh of fixed dimensions is transformed in the vtx shader to cover each selected node area in the world space, and vtx heights are displaced using texture fetches, thus forming the representation of the particular terrain patch"

1. **Fixed 크기 grid mesh 하나**만 미리 생성, GPU 메모리에 딱 한번 올라간 이후 모든 노드 재사용!
2. Selected Node(=selected List에 담긴 Position, LOD level)마다 grid를 **VS**에서 노드가 cover하는 world position 영역에 맞게 scale,move 시긴다.
   -> 노드 크기 크면 Grid 넓게, 작으면 좁게 배치
3. 배치된 VTX마다 Heightmap texture를 sampling해서 x,z 위치 높이가 얼마다 읽어와 y displace!
- **patch** : 선택되 노드 하나가 담당하는 지형 영역

`16x16`~ `128x128`은 모든 노드에 동일히 적용, 런타임에 변경될 수 있다. (Reflection, LowQuality등과 같은) 낮은 detail 필요한 효과를 위해 성능 아낄 수 있음
++ LOD system과 Grid Resolution은 **서로 독립적인 축**


### Morph implement

<table width="90%" style="table-layout: fixed; border-collapse: collapse; border: none;"> <tr style="border: none;"> <td width="100%" style="text-align: center; border: none; padding: 15px;"> <img src="/assets/postimg/TerrainOpti/CDLOD/Morph_Figure.png" alt="VS003" style="width: 100%; max-width: 100%; height: auto;"> <br><strong>Morph Figure</strong> </td>  </tr> </table>
High res의 가운데 줄의 정점이 사라짐 ->**홀수 index vtx**들이 이웃 **짝수 정점 위치**로 합쳐지도록 하는 concept


``` hlsl
// gridPos : [0,1] nomalize된 mesh position
// morphk : morph value
const float2 g_gridDim = float2(64, 64);
float2 morphVertex(float2 GridPos, float2 vertex, float morphK)
{ 
	float2 fracPart = frac(gridPos.xy * g_gridDim.xy * 0.5) * 2.0/ g_gridDim.xy;
	//
	
	return vertex.xy - fracPart * g_quadScale.xy * morphK;
	// vtx 이동 fracpart랑 정규화 푼것, morphK만큼
}
```


`gridDim * 0.5` : 정수 grid index 절반값
- 짝수 index  i=2 : `2 * 0.5 = 1.0(정수)`
- 홀수 index  i = 1 : `1 * 0.5 = 0.5(소수)`

frac으로 감싸서, 홀수만 정보 가질 수있도록

 `*2.0 / g_gridDimx` 으로 정규화 다시 돌림


-**개별 vtx**(홀수 index)만 짝수 위치로 끌려가도록, 그 이후 **merge**
