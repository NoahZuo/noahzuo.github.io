---
title: Tool Tips About UE4 Cooking
tags: 
 - UE4
category: UE4
date: 2021-09-12 15:10:03
updated: 2021-11-16 14:48:28
description: This article simply records some issues about UE4 cooking that I encountered these days. 
---

# How To Debug UE4 Cooking Stage? 

When an error is triggered during cooking, the break point would not be hit in your visual studio because another `UE4Editor` process is invoked to run cooking command. 



Without help of Visual Studio, it must be very hard to address and fix your cooking error. 



In Visual Studio, right-click on your project->Properties: 

![image_001](image_001.png)

Set the configuration to `Development_Editor` or `Debug_Editor`, and you can find the `Command Arguments` in the `Debugging` tab: 

![img](image_002.png)

Usually the default value should be `"$(SolutionDir)$(ProjectName).uproject" -skipcompile`. And change it to `"$(SolutionDir)$(ProjectName).uproject" -run=Cook -TargetPlatform=WindowsNoEditor -unversioned -CookAll`: 

![image_001](image_003.png)

And it's done. Hit `Local Windows Debugger`, and start debuging your cooking stage! 



# How To Cook A Single Asset? 

It seems that the `HotPatcher` plugin has already provided us a `OnCookPlatform` function, which cooks our selected assets in `BrowserContent`: 

```cpp
void FHotPatcherEditorModule::OnCookPlatform(ETargetPlatform Platform)
{
        TArray<FAssetData> AssetsData = GetSelectedAssetsInBrowserContent();
        TArray<UPackage*> AssetsPackage;

        for(const auto& AssetData:AssetsData)
        {
                AssetsPackage.AddUnique(AssetData.GetPackage());
                UE_LOG(LogHotPatcher,Log,TEXT("Cook Package %s Platform %s"),*AssetData.PackagePath.ToString(),*UFlibPatchParserHelper::GetEnumNameByValue(Platform));
        }
        FString CookedDir = FPaths::ConvertRelativePathToFull(FPaths::Combine(FPaths::ProjectSavedDir(),TEXT("Cooked")));
        FString CookForPlatform = UFlibPatchParserHelper::GetEnumNameByValue(Platform);
        TArray<FString> CookForPlatforms{CookForPlatform};
        
        for(const auto& AssetPacakge:AssetsPackage)
        {
                FString CookedSavePath = UFlibHotPatcherEditorHelper::GetCookAssetsSaveDir(CookedDir,AssetPacakge, CookForPlatform);
                
            bool bCookStatus = UFlibHotPatcherEditorHelper::CookPackage(AssetPacakge,CookForPlatforms,CookedDir);
                auto Msg = FText::Format(
            LOCTEXT("CookAssetsNotify", "Cook Platform {1} for {0} {2}!"),
            UKismetTextLibrary::Conv_StringToText(AssetPacakge->FileName.ToString()),
            UKismetTextLibrary::Conv_StringToText(UFlibPatchParserHelper::GetEnumNameByValue(Platform)),
            bCookStatus ? UKismetTextLibrary::Conv_StringToText(TEXT("Successfuly")):UKismetTextLibrary::Conv_StringToText(TEXT("Faild"))
            );
                SNotificationItem::ECompletionState CookStatus = bCookStatus ? SNotificationItem::CS_Success:SNotificationItem::CS_Fail;
                UFlibHotPatcherEditorHelper::CreateSaveFileNotify(Msg,CookedSavePath,CookStatus);
        }
        
}
```

We can easily create an `FAutoConsoleCommand` to manually achieve this feature. 



# How To Get Rid Of Property When Cooking? 

We might want to get rid of some properties during cooking stage. For example, `RawAnimationData` is supposed to be stripped for cooked builds in consideration of memory.  

## For A Non-UPROPERTY Member

It might get really complicated to manage since we need to handle `PostLoad` and `Serialize` function manually, but we can get more flexibility. 

If you want to remove this property, use `FPlatformProperties::RequiresCookedData()` function to determine if you need it or not. 

## For A UPROPERTY Member

In a more complicated situation if you still need to handle this property during cooking stage, and frankly speaking, this is exactly what I needed these days. We might need a hook to get rid of this property. 

Sadly, I cannot tell you what precisely I needed to do because of secrecy. 

We can remove its content in `UObject::PreSave` function. 

In a cooked build, `PreSave` function would be called before save-serializing happens. That is a wonderful hook for you to modify this data. 