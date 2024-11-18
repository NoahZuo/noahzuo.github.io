---
title: How to Create Asset Right Click Menu In UE4
tags: 
 - UE4
category: UE4
date: 2020-08-28 15:45:12
updated: 2020-08-28 17:30:28
description: This article tells you how to create asset right click menu in UE4. 

---

It would be wonderful if we can create our own asset menu to process assets. But it seems not that easy to achieve that. I always miss Unity when it comes to creation of a customized editor tool. 

# Prerequisites
First of all, it is recommended to  you are guranteed to have these menus setup in the `StartModule` function: 

```cpp
void F***Module::StartupModule()
{
    // Set up your menus here.
}
```

It's easy to know that we should release those menus in the `ShutdownModule` function: 

```cpp
void F***Module::ShutdownModule()
{
    // Release your menus here. 
}
```

The `Content Browser` module loaded if you want to create customized right click menu. As a result, we need to load this module by: 

```cpp
FContentBrowserModule& ContentBrowserModule = FModuleManager::LoadModuleChecked<FContentBrowserModule>(TEXT("ContentBrowser"));
```

# Core Logic

## Menu Extender

For a right click function in the content browser, a menu extender should be created and added to the current menu extender array by: 

```cpp
TArray<FContentBrowserMenuExtender_SelectedAssets>& CBMenuAssetExtenderDelegates = ContentBrowserModule.GetAllAssetViewContextMenuExtenders();
	CBMenuAssetExtenderDelegates.Add(FContentBrowserMenuExtender_SelectedAssets::CreateStatic(&YourOnExtendAssetSelectionMenu));
```

`YourOnExtendAssetSelectionMenu` is a static function that accepts a `const TArray<FAssetData>&` and returns a `TSharedRef<FExtender>`: 

```cpp
static TSharedRef<FExtender> YourOnExtendAssetSelectionMenu(const TArray<FAssetData>& SelectedAssets)
{
	TSharedRef<FExtender> Extender = MakeShared<FExtender>();
	Extender->AddMenuExtension(
		"CommonAssetActions",
		EExtensionHook::After,
		nullptr,
		FMenuExtensionDelegate::CreateStatic(&YourAssetExtenderFunc, SelectedAssets)
	);
	return Extender;
}
```

Here another function has just came to us. We still need an `AssetExtenderFunction` which accepts an `FMenuBuilder&` and an array of `FAssetData`. We add menu entry here to eventually add a button by: 

```cpp
static void YourAssetExtenderFunc(FMenuBuilder& MenuBuilder, const TArray<FAssetData> SelectedAssets)
{
    MenuBuilder.BeginSection("Your Asset Context", LOCTEXT("ASSET_CONTEXT", "Your Asset Context"));
    {
        // Add Menu Entry Here
    }
    MenuBuilder.EndSection();
}
```

## Filtering the Selected Asset

We always need to add some menu to the specific asset. For example, `Find Skeleton` button would only appear in the right-click menu for a `Animation Sequence Base` asset: 

![image-20200829164232753](image-20200829164232753.png)

We can filter asset data type in `YourAssetExtenderFunc` like this: 

```cpp
for (auto AssetIt = SelectedAssets.CreateConstIterator(); AssetIt; ++AssetIt)
{
    const FAssetData& Asset = *AssetIt;
    
    if (!Asset.IsRedirector() && Asset.AssetClass != NAME_Class && !(Asset.PackageFlags & PKG_FilterEditorOnly))
    {
        if (Asset.GetClass()->IsChildOf(UAnimSequenceBase::StaticClass()))
		{
            // ... 
		}
    }
}
```

## Add Menu Entry

Simply use a Lambda function for this like: 

```cpp
MenuBuilder.AddMenuEntry(
	LOCTEXT("ButtonName", "ButtonName"),
	LOCTEXT("Button ToolTip", "Button ToolTip"),
	FSlateIcon(FLinterStyle::GetStyleSetName(), "Linter.Toolbar.Icon"),
	FUIAction(FExecuteAction::CreateLambda([SelectedAssets]()
	{
		// Do work here. 
	})),
	NAME_None,
	EUserInterfaceActionType::Button);
```

And that should work. 











