---
layout: post
published: false
title: Unreal DataAsset vs PrimaryDataAsset
subtitle: Constraint
date: 2026-06-24 19:37:00 +0900
description: Unreal Code
categories:
  - Engine
tags:
  - UnrealClass
  - Unreal
---

Unreal cpp의 Data Structure나 Delegate 패턴,  Capture lambda 등을 코드를 뜯어보기로 했다.

``` cpp
void UGameFeatureData::InitializeHierarchicalPluginIniFiles(cosnt FString& PluginInstalledFilename) const
```
Lyra예제파일에서의 GameFeatureData : public UPrimaryDataAsset

모든 코드를 뜯어 설명하기보다 중간 모르는 키워드를 정리한다.

---

### InitializeHierarchicalPluginIniFiles 전체 함수 구조

Get

### Class

#### UDeviceProfileManager?

``` cpp
class UMyData : public UDataAsset
{
	UPROPERTY()
	int32 Damage;
}
```
데이터 묶음 파일, 별 다른 기능 없음

#### FPlatformProperties

``` cpp
class UMyData : public UPrimaryDataAsset
{
	UPROPERTY()
	int32 Damage;
	
public:
	//요놈 핵심
	virtual FPrimaryAssetId GetPrimaryAssetId() const override; 
}

```

| 기능                  | DataAsset | Primary Data Asset |
| ------------------- | --------- | ------------------ |
| 데이터 저장              | ✅         | ✅                  |
| 에디터에서 편집            | ✅         | ✅                  |
| 다른 에셋이 직접 참조        | ✅         | ✅                  |
| **AssetManager 등록** | ❌         | ✅                  |
| **이름으로 검색/로드**      | ❌         | ✅                  |
| **번들 단위 로드/언로드**    | ❌         | ✅                  |
| **쿠킹/패키징 규칙 지정**    | ❌         | ✅                  |

## 2. 접근 : DataAsset vs PrimaryDataAsset
#### Asset
``` cpp
UMyData* Data = LoadObject<UMyData>(
	nullptr,TEXT("/Game/Asset/Weapons/DA_Sword.DA_Sword") // 경로로 직접 찾기!
	);
		
```
반드시 직접 하드코딩 / 다른 에셋이 참조하고 있어야함!

#### Primary Asset
``` cpp
FPrimaryAssetId Id( "MyDataType", "MyData");

UAssetManager::Get().LoadPrimaryAsset(
	Id, TArray<FName>(), FStreamableDelegate::CreateLambda([Id]()
	{
		UMyData* Data = UAssetManager::Get().GetPrimaryAssetObject<UMyData>(Id);
	})
);
```
AssetManager를 통해 ID로 요청 가능!

> AssetManager 조회-> 실제 패키지 경로 획득 -> 로드

: Game Ability System에서의 Ability, GameplayEffect, CharacterClassData, ItemData같은 것들이
Primary Data Asset을 많이 쓰는 이유!



## 3. 선택 : DataAsset vs PrimaryDataAsset

#### DataAsset
- 다른 BP / Asset이 직접 들고 있을때
- 작은 규모의 프로젝트
- EX) Character BP가 멤버 변수를 들고있는 스탯 데이터

#### PrimaryDataAsset
- 런타임에 동적으로 Load/UnLoad
- 어떤 Asset이 존재하는지 목록 검색해야할때
- EX) 아이템 100종류를 필요할 때만 꺼내 쓰는 경우



#### 최종 코드
``` cpp
void UGameFeatureData::InitializeHierarchicalPluginIniFiles(const FString& PluginInstalledFilename) const
{
	UDeviceProfileManager& DeviceProfileManager = UDeviceProfileManager::Get();

	FString PlatformName = FPlatformProperties::IniPlatformName();

#if ALLOW_OTHER_PLATFORM_CONFIG
	const UDeviceProfile* PreviewDeviceProfile = DeviceProfileManager.GetPreviewDeviceProfile();
	if (PreviewDeviceProfile)
	{
		PlatformName = PreviewDeviceProfile->ConfigPlatform.IsEmpty() ? PreviewDeviceProfile->DeviceType : PreviewDeviceProfile->ConfigPlatform;
	}
#endif

	FString PluginInstalledStandardFilename = PluginInstalledFilename;
	if (!FPaths::IsRelative(PluginInstalledStandardFilename))
	{
		FPaths::MakeStandardFilename(PluginInstalledStandardFilename);
	}

	const FString PluginName = FPaths::GetBaseFilename(PluginInstalledStandardFilename);
	const FString PlatformExtensionDir = FPaths::ProjectPlatformExtensionDir(*PlatformName);
	const FString EngineConfigDir = FPaths::EngineConfigDir();
	const FString PluginConfigDir = FPaths::GetPath(PluginInstalledStandardFilename) / TEXT("Config/");
	const FString PluginPlatformConfigDir = FPaths::Combine(PluginConfigDir, PlatformName);
	const FString PluginPlatformExtensionDir = FPaths::GetPath(PluginInstalledStandardFilename).Replace(*FPaths::ProjectDir(), *PlatformExtensionDir) / TEXT("Config");

	// We're going to test a lot of paths, so only do it if some config actually exists
	if (!FPaths::DirectoryExists(PluginConfigDir) && !FPaths::DirectoryExists(PluginPlatformConfigDir) && !FPaths::DirectoryExists(PluginPlatformExtensionDir))
	{
		return;
	}

	const bool bIsBaseIniName = false;
	const bool bForceReloadFromDisk = false;
	const bool bWriteDestIni = false;
	const bool bCreateDeviceProfiles = CVarAllowRuntimeDeviceProfiles->GetBool();

	struct FIniLoadingParams
	{
		FIniLoadingParams(const FString& InName, bool bInUsePlatformDir = false, bool bInCreateDeviceProfiles = false)
			: Name(InName), bUsePlatformDir(bInUsePlatformDir), bCreateDeviceProfiles(bInCreateDeviceProfiles)
		{}

		FString Name;
		bool bUsePlatformDir;
		bool bCreateDeviceProfiles;
	};

	// Engine.ini and DeviceProfiles.ini will also support platform extensions
	TArray<FIniLoadingParams> IniFilesToLoad = { FIniLoadingParams(TEXT("Input")),
		FIniLoadingParams(TEXT("Game")), FIniLoadingParams(TEXT("Game"), true),
		FIniLoadingParams(TEXT("Engine")), FIniLoadingParams(TEXT("Engine"), true),
#if UE_EDITOR
		FIniLoadingParams(TEXT("Editor")),
#endif
		FIniLoadingParams(TEXT("DeviceProfiles"), false, bCreateDeviceProfiles),
		FIniLoadingParams(TEXT("DeviceProfiles"), true, bCreateDeviceProfiles)
	};

	// Create overridden device profiles for each matching rule in config
	auto InsertRuntimeDeviceProfilesIntoConfig = [&](FConfigFile& PluginConfig, const FConfigFile& ExistingConfig)
	{
		TMap<FString, FConfigSection> ConfigSectionsToAdd;

		TArray<FString> ResultingProfiles;

		for (auto& Section : AsConst(PluginConfig))
		{
			FString RuleName, ParentClass;
			if (Section.Key.Split(TEXT(" "), &RuleName, &ParentClass))
			{
				// Early reject anything that's not handled in here
				const bool bIsRuntimeDeviceProfileRule = (ParentClass == "RuntimeDeviceProfileRule");
				const bool bIsDeviceProfileFragment = (ParentClass == UDeviceProfileFragment::StaticClass()->GetName());
				if (!(bIsRuntimeDeviceProfileRule || bIsDeviceProfileFragment))
				{
					continue;
				}

				// Check the existing config because at this point in time it has already been hotfixed from the base empty config
				// Those CVars are also always without a +-. prefix because they're the result of the hotfix applied to the empty config.
				// @todo: we cannot use hotfixes to remove CVars because HF happens way before GFPs have a chance to load.
				// The hotfix process is destructive and doesn't leave us an opportunity to read the hotfix delta when loading GFPs.
				TArray<FConfigValue> HotfixCVars;
				if (const FConfigSection* HotfixSection = ExistingConfig.FindSection(Section.Key))
				{
					HotfixSection->MultiFind("CVars", HotfixCVars);
				}

				// Extract key-value pairs for CVars and FragmentIncludes, keeping the +-. prefix
				TMultiMap<FName, FConfigValue> PluginCVars;
				TMultiMap<FName, FConfigValue> FragmentIncludes;
				for (const auto& Entry : Section.Value)
				{
					const FString& EntryKey = Entry.Key.ToString();
					if (EntryKey.RightChop(1).StartsWith("CVars"))
					{
						PluginCVars.Add(Entry.Key, Entry.Value);
					}
					else if (EntryKey.RightChop(1).StartsWith("FragmentIncludes"))
					{
						FragmentIncludes.Add(Entry.Key, Entry.Value);
					}
				}

				// Check if a CVar should be either included, or removed for a new hotfix value
				auto ShouldKeepCVar = [&HotfixCVars](const FString& Key, FString& Value) -> bool
				{
					for (const FConfigValue& HotfixCVarData : HotfixCVars)
					{
						FString HotfixCVarKey, HotfixCVarValue;
						if (HotfixCVarData.GetValue().Split(TEXT("="), &HotfixCVarKey, &HotfixCVarValue) && HotfixCVarKey == Key && HotfixCVarValue != Value)
						{
							Value = HotfixCVarValue;
							return false;
						}
					}
					return true;
				};

				// Process new runtime device profile
				if (bIsRuntimeDeviceProfileRule)
				{
					UE_LOG(LogGameFeatures, Log, TEXT("Game feature '%s' found runtime device profile rule %s"), *PluginName, *RuleName);

					// Extract metadata
					const FConfigValue* ParentProfileName = Section.Value.Find("ParentProfileName");
					const FConfigValue* ProfileSuffix = Section.Value.Find("ProfileSuffix");

					if (ParentProfileName && ProfileSuffix)
					{
						// We need to load all candidate device profiles here or else we won't be able to create a child for them
						TArray<FString> LoadableProfileNames = DeviceProfileManager.GetLoadableProfileNames(*PlatformName);
						for (const FString& ProfileName : LoadableProfileNames)
						{
							DeviceProfileManager.FindProfile(ProfileName, true, *PlatformName);
						}

						for (const UDeviceProfile* Profile : DeviceProfileManager.Profiles)
						{
							// Check if one of the parents for this profile is the one this rule applies to
							bool bProfileHasCompatibleParent = false;
							const UDeviceProfile* CurrentProfile = Profile;
							do
							{
								bProfileHasCompatibleParent = CurrentProfile->GetName() == ParentProfileName->GetValue();
								CurrentProfile = CurrentProfile->GetParentProfile();
							} while (!bProfileHasCompatibleParent && CurrentProfile != nullptr);

							// Create the config for a runtime profile
							if (bProfileHasCompatibleParent && !Profile->GetName().EndsWith(ProfileSuffix->GetValue()))
							{
								// Ignore duplicates: only the first match in a given config file will be accepted
								const FString FinalProfileName = Profile->GetName() + ProfileSuffix->GetValue();
								if (ResultingProfiles.Contains(FinalProfileName))
								{
									UE_LOG(LogGameFeatures, Log, TEXT("Ignoring profile %s that has already been overriden as %s"), *Profile->GetName(), *FinalProfileName);
									continue;
								}

								FConfigSection RuntimeProfile;
								RuntimeProfile.Add("DeviceType", PlatformName);
								RuntimeProfile.Add("BaseProfileName", FConfigValue(Profile->GetName()));

								UE_LOG(LogGameFeatures, Log, TEXT("Creating override for base profile %s"), *Profile->GetName());

								// Inject the parent's matched fragments into the config, if any
								if (Profile->GetName().Contains("MatchedFragments"))
								{
									FString MatchingRulesSectionName = Profile->GetName() + TEXT(" ") + UDeviceProfile::StaticClass()->GetName();
									FString MatchingRulesArrayName = TEXT("MatchingRules");
									TArray<FString> MatchingRulesArray;

#if ALLOW_OTHER_PLATFORM_CONFIG
									FConfigCacheIni* PlatformConfigSystem = FConfigCacheIni::ForPlatform(*PlatformName);
#else
									FConfigCacheIni* PlatformConfigSystem = GConfig;
#endif

									PlatformConfigSystem->GetArray(*MatchingRulesSectionName, *MatchingRulesArrayName, MatchingRulesArray, GDeviceProfilesIni);
									UE_LOG(LogGameFeatures, Log, TEXT("Found %d fragment matching rules"), MatchingRulesArray.Num());

									for (const FString& Rule : MatchingRulesArray)
									{
										RuntimeProfile.Add("+MatchingRules", FConfigValue(Rule));
									}
								}

								// Add fragment includes
								for (const auto& FragmentInclude : FragmentIncludes)
								{
									RuntimeProfile.Add(FragmentInclude.Key, FConfigValue(FragmentInclude.Value.GetValue()));
								}

								// Add direct CVars
								for (const auto& CVar : PluginCVars)
								{
									FString CVarKey, CVarValue;
									if (CVar.Value.GetValue().Split(TEXT("="), &CVarKey, &CVarValue) && ShouldKeepCVar(CVarKey, CVarValue))
									{
										UE_LOG(LogGameFeatures, Log, TEXT(" Found CVar: %s=%s"), *CVarKey, *CVarValue);
										RuntimeProfile.Add(CVar.Key, FConfigValue(CVarKey + "=" + CVarValue));
									}
								}

								// Add hotfix CVars
								for (const auto& CVar : HotfixCVars)
								{
									FString CVarKey, CVarValue;
									if (CVar.GetValue().Split(TEXT("="), &CVarKey, &CVarValue) && !PluginCVars.Contains(FName(*CVarKey)))
									{
										UE_LOG(LogGameFeatures, Log, TEXT(" Added CVar: %s=%s"), *CVarKey, *CVarValue);
										RuntimeProfile.Add("+CVars", FConfigValue(CVarKey + "=" + CVarValue));
									}
								}

								ConfigSectionsToAdd.Add(FinalProfileName + TEXT(" ") + UDeviceProfile::StaticClass()->GetName(), RuntimeProfile);
								ResultingProfiles.Add(FinalProfileName);
							}
						}
					}
					else
					{
						UE_LOG(LogGameFeatures, Warning, TEXT("Game feature '%s' has invalid runtime device profile with parent %s, suffix %s, %d CVars, %d fragments"),
							*PluginName, ParentProfileName ? *ParentProfileName->GetValue() : TEXT("null"), ProfileSuffix ? *ProfileSuffix->GetValue() : TEXT("null"),
							FragmentIncludes.Num(), PluginCVars.Num());
					}
				}

				// Hotfix device profile fragments
				else if (bIsDeviceProfileFragment)
				{
					UE_LOG(LogGameFeatures, Log, TEXT("Game feature '%s' found device profile fragment %s"), *PluginName, *RuleName);

					if (HotfixCVars.Num() > 0)
					{
						// Update existing CVars
						for (const auto& CVar : PluginCVars)
						{
							FString PluginCVarKey, PluginCVarValue;
							if (CVar.Value.GetValue().Split(TEXT("="), &PluginCVarKey, &PluginCVarValue) && !ShouldKeepCVar(PluginCVarKey, PluginCVarValue))
							{
								UE_LOG(LogGameFeatures, Log, TEXT(" Removed CVar: %s"), *CVar.Value.GetValue());
								PluginConfig.RemoveFromSection(*Section.Key, CVar.Key, CVar.Value.GetValue());
							}
							else
							{
								UE_LOG(LogGameFeatures, Log, TEXT(" Kept CVar: %s=%s"), *PluginCVarKey, *PluginCVarValue);
							}
						}

						// Add new hotfix CVars
						for (const auto& CVar : HotfixCVars)
						{
							FString HotfixCVarKey, HotfixCVarValue;
							if (CVar.GetValue().Split(TEXT("="), &HotfixCVarKey, &HotfixCVarValue) && !PluginCVars.Contains(FName(*HotfixCVarKey)))
							{
								UE_LOG(LogGameFeatures, Log, TEXT(" Added CVar: %s=%s"), *HotfixCVarKey, *HotfixCVarValue);
								PluginConfig.AddToSection(*Section.Key, "+CVars", HotfixCVarKey + "=" + HotfixCVarValue);
							}
						}
					}
				}
			}
		}

		PluginConfig.Append(ConfigSectionsToAdd);
	};

	// Create device profiles for this plugin from config
	auto LoadDeviceProfilesFromConfig = [&](const FConfigFile& Config)
	{
		for (TPair<const FString&, const FConfigSection&> Section : Config)
		{
			FString ProfileName;
			FString ParentClass;
			if (Section.Key.Split(TEXT(" "), &ProfileName, &ParentClass) && ParentClass == UDeviceProfile::StaticClass()->GetName())
			{
				const FConfigValue* DeviceType = Section.Value.Find("DeviceType");
				if (DeviceType)
				{
					UE_LOG(LogGameFeatures, Log, TEXT("Game feature '%s' adding new device profile %s"), *PluginName, *ProfileName);
					DeviceProfileManager.CreateProfile(ProfileName, DeviceType->GetValue(), FString(), *PlatformName);
				}
			}
		}
	};

	// @todo: Likely we need to track the diffs this config caused and/or store versions/layers in order to unwind settings during unloading/deactivation
	for (const FIniLoadingParams& Ini : IniFilesToLoad)
	{
		const FString PluginIniName = Ini.bUsePlatformDir ? (PlatformName + PluginName + Ini.Name) : PluginName + Ini.Name;

		FString ConfigDirectory;
		if (Ini.bUsePlatformDir)
		{
			// We'll look first in the platform extension directory, then in the plugin's platform directory
			if (FPaths::FileExists(FPaths::Combine(PluginPlatformExtensionDir, PluginIniName + ".ini")))
			{
				ConfigDirectory = PluginPlatformExtensionDir;
			}
			else
			{
				ConfigDirectory = PluginPlatformConfigDir;
			}
		}
		else
		{
			ConfigDirectory = PluginConfigDir;
		}
		ConfigDirectory += TEXT("/");

		// @note: Loading the INI in this manner in order to have a record of relevant sections that were changed so that affected objects can be reloaded. By virtue of how
		// this is parsed (standalone instead of being treated as a combined diff), the actual data within the sections will likely be incorrect. As an example, users adding
		// to an array with the "+" syntax will have the "+" incorrectly embedded inside the data in the temp FConfigFile. It's properly handled in the Combine() below where the
		// actual INI changes are computed.
		FConfigFile Config;
		if (FConfigCacheIni::LoadExternalIniFile(Config, *PluginIniName, *EngineConfigDir, *ConfigDirectory, bIsBaseIniName, nullptr, bForceReloadFromDisk, bWriteDestIni) && (Config.Num() > 0))
		{
			UE_LOG(LogGameFeatures, Log, TEXT("Game feature '%s' loaded config file %s"), *PluginName, *PluginIniName);

			// Need to get the in-memory config filename, the on disk one is likely not up to date
			FString IniFile = GConfig->GetConfigFilename(*Ini.Name);

			// Ensure we push new device profile config to the appropriate config branch - GConfig could be Windows while we're previewing a console
			FConfigFile* ExistingConfig = nullptr;
#if ALLOW_OTHER_PLATFORM_CONFIG
			if (Ini.bCreateDeviceProfiles && Ini.bUsePlatformDir && !FPlatformProperties::RequiresCookedData())
			{
				FConfigCacheIni* PlatformConfigSystem = FConfigCacheIni::ForPlatform(*PlatformName);
				ExistingConfig = PlatformConfigSystem->FindConfigFile(GDeviceProfilesIni);
			}
#endif

			if (ExistingConfig == nullptr)
			{
				ExistingConfig = GConfig->FindConfigFile(IniFile);
			}

			if (ExistingConfig)
			{
				if (Ini.bCreateDeviceProfiles)
				{
					InsertRuntimeDeviceProfilesIntoConfig(Config, *ExistingConfig);
				}

				FString ConfigAsString;
				Config.WriteToString(ConfigAsString, PluginIniName);

				// @todo: Might want to consider modifying the engine level's API here to allow for a combination that yields affected
				// sections and/or optionally just does the reload itself. This route is less efficient than it needs to be, resulting in parsing twice, 
				// once above and once in the Combine() call. Using Combine() here specifically so that special INI syntax (+, ., etc.) is parsed correctly.
				const FString PluginIniPath = FString::Printf(TEXT("%s%s.ini"), *ConfigDirectory, *PluginIniName);
				ExistingConfig->CombineFromBuffer(ConfigAsString, PluginIniPath);

#if ALLOW_INI_OVERRIDE_FROM_COMMANDLINE
				FConfigFile::OverrideFromCommandline(ExistingConfig, Ini.Name);
#endif // ALLOW_INI_OVERRIDE_FROM_COMMANDLINE

				if (Ini.bCreateDeviceProfiles)
				{
					LoadDeviceProfilesFromConfig(Config);
				}
				else
				{
					ReloadConfigs(Config);
				}
			}
		}
	}
}
```