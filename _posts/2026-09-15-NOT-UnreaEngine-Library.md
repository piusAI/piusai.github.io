---
layout: post
published: false
title: UE library For Tools
subtitle: Constraint
date: 2026-08-26 19:37:00 +0900
description: Unreal DataAsset vs PrimaryDataAsset
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---

활용되는 Library들을 저장 해두려한다.

## Library, Asset 관련

# Unreal Editor Utility Library 정리

## Asset 관련

| Library / Namespace     | 함수                               | 시그니처                                                                                                         | 설명                                   | 소스 링크                                                                                                                                                          |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ObjectTools`           | `DeleteAssets`                   | `int32 DeleteAssets(const TArray<FAssetData>& AssetsToDelete, bool bShowConfirmation = true)`                | 선택한 에셋들을 삭제. 반환값은 실제 삭제된 개수          | [ObjectTools](https://github.com/EpicGames/UnrealEngine/blob/16d75d84714512edfb744e1fd0a59e9c74d57873/Engine/Source/Editor/UnrealEd/Public/ObjectTools.h#L374) |
| `UEditorAssetLibrary`   | `FindPackageReferencersForAsset` | `TArray<FString> FindPackageReferencersForAsset(const FString& AssetPath, bool bLoadAssetsToConfirm = true)` | 해당 에셋을 참조하는 패키지 목록 반환. 비어있으면 미사용 에셋  |                                                                                                                                                                |
| `UEditorUtilityLibrary` | `GetSelectedAssetData`           | `TArray<FAssetData> GetSelectedAssetData()`                                                                  | 콘텐츠 브라우저에서 현재 선택된 에셋들의 FAssetData 반환 |                                                                                                                                                                |
|                         |                                  |                                                                                                              |                                      |                                                                                                                                                                |
|                         |                                  |                                                                                                              |                                      |                                                                                                                                                                |

## Library 관련

| Library / Namespace | 함수 | 시그니처 | 설명 | 소스 링크 |
|---|---|---|---|---|
|  |  |  |  |  |
|  |  |  |  |  |

