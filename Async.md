# Async

Do Work in other Thread within a Callback(Delegate in UE)

## Create Blueprint Async Node
    1. Inherit from UBlueprintAsyncActionBase
    2. Add to Root for provide from GC
    3. Define Nodes' Method
```c++
    UMyBlueprintAsyncActionBase* UMyBlueprintAsyncActionBase::LoadImageFromDiskAsync(const FString _Path, const FString _Key)
    {
        UMyBlueprintAsyncActionBase* BlueprintNode = NewObject<UMyBlueprintAsyncActionBase>();
        BlueprintNode->Start(_Path, _Key, BlueprintNode);
        return BlueprintNode;
    }

```
    4. Create a Async Task Thread

``` c++
e.g.

Async(EAsyncExecution::ThreadPool, [_path,_key,ImageAsync,ImageWrapper]()
{
    UTexture2D* Texture = ImageAsync->LoadTexture2D(_path,_key,ImageWrapper);
    FObImageLoadComplete OnComplete = ImageAsync->ImageLoadComplete;
    OnComplete.Broadcast(Texture, _key);
});

```

### Full Code
<font color="Yellow">MyBlueprintAsyncActionBase.h</font>
```c++
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintAsyncActionBase.h"
#include "MyBlueprintAsyncActionBase.generated.h"

/**
 * 
 */

DECLARE_DYNAMIC_MULTICAST_DELEGATE_TwoParams(FObImageLoadComplete, UTexture2D*, Texture, FString, Key);

class IImageWrapper;

UCLASS()
class PAKTEST_API UMyBlueprintAsyncActionBase : public UBlueprintAsyncActionBase
{
	GENERATED_BODY()
public:
	UMyBlueprintAsyncActionBase();
	
	UPROPERTY(BlueprintAssignable)
	FObImageLoadComplete ImageLoadComplete;

public:

	UFUNCTION(BlueprintCallable, meta = (BlueprintInternalUseOnly = "true", WorldContext = "WorldContextObject"))
	static UMyBlueprintAsyncActionBase* LoadImageFromDiskAsync(const FString _Path, const FString _Key);

	void Start(const FString _path, const FString _key, UMyBlueprintAsyncActionBase* ImageAsync);

	TSharedPtr<IImageWrapper> GetImageWrapperByExtention(const FString InImagePath);

	UTexture2D* LoadTexture2D(const FString _path, const FString _key, TSharedPtr<IImageWrapper> ImageWrapper);

	
};

```

<font color="Yellow">MyBlueprintAsyncActionBase.cpp</font>

```c++
// Fill out your copyright notice in the Description page of Project Settings.


#include "MyBlueprintAsyncActionBase.h"

#include "IImageWrapper.h"
#include "IImageWrapperModule.h"

UMyBlueprintAsyncActionBase::UMyBlueprintAsyncActionBase()
{
	if(HasAnyFlags(RF_ClassDefaultObject) == false)
	{
		AddToRoot();
	}
}


// UMyBlueprintAsyncActionBase::UMyBlueprintAsyncActionBase(const FObjectInitializer& ObjectInitializer)
// : Super(ObjectInitializer)
// {
// 	if(HasAnyFlags(RF_ClassDefaultObject) == false)
// 	{
// 		AddToRoot();
// 	}
// }

UMyBlueprintAsyncActionBase* UMyBlueprintAsyncActionBase::LoadImageFromDiskAsync(const FString _Path, const FString _Key)
{
	UMyBlueprintAsyncActionBase* BlueprintNode = NewObject<UMyBlueprintAsyncActionBase>();
	BlueprintNode->Start(_Path, _Key, BlueprintNode);
	return BlueprintNode;
}

void UMyBlueprintAsyncActionBase::Start(const FString _path, const FString _key, UMyBlueprintAsyncActionBase* ImageAsync)
{
	TSharedPtr<IImageWrapper> ImageWrapper = ImageAsync->GetImageWrapperByExtention(".jpeg");
	
	Async(EAsyncExecution::ThreadPool, [_path,_key,ImageAsync,ImageWrapper]()
	{
		UTexture2D* Texture = ImageAsync->LoadTexture2D(_path,_key,ImageWrapper);
		FObImageLoadComplete OnComplete = ImageAsync->ImageLoadComplete;
		OnComplete.Broadcast(Texture, _key);
	});
	
}

TSharedPtr<IImageWrapper> UMyBlueprintAsyncActionBase::GetImageWrapperByExtention(const FString InImagePath)
{
	IImageWrapperModule& ImageWrapperModule = FModuleManager::LoadModuleChecked<IImageWrapperModule>(FName("ImageWrapper"));

	if(InImagePath.EndsWith(".jpg") || InImagePath.EndsWith(".jpeg"))
	{
		return ImageWrapperModule.CreateImageWrapper(EImageFormat::JPEG);
	}
	return nullptr;
}

UTexture2D* UMyBlueprintAsyncActionBase::LoadTexture2D(const FString _path, const FString _key, TSharedPtr<IImageWrapper> ImageWrapper)
{
	UTexture2D* Texture = nullptr;

	if(!FPlatformFileManager::Get().GetPlatformFile().FileExists(*_path))
	{
		return nullptr;
	}
	TArray<uint8> RawFileData;
	if(!FFileHelper::LoadFileToArray(RawFileData, *_path))
	{
		return nullptr;
	}

	bool IsNull = ImageWrapper->SetCompressed(RawFileData.GetData(), RawFileData.Num());
	if(ImageWrapper.IsValid() && ImageWrapper->SetCompressed(RawFileData.GetData(), RawFileData.Num()))
	{
		TArray<uint8> UncompressedRGBA ;
		if(ImageWrapper->GetRaw(ERGBFormat::BGRA, 8, UncompressedRGBA))
		{
			Texture = UTexture2D::CreateTransient(ImageWrapper->GetWidth(), ImageWrapper->GetHeight(), PF_B8G8R8A8);
			if(Texture)
			{
				void* TextureData = Texture->GetPlatformData()->Mips[0].BulkData.Lock(LOCK_READ_WRITE);
				FMemory::Memcpy(TextureData, UncompressedRGBA.GetData(), UncompressedRGBA.Num());
				Texture->GetPlatformData()->Mips[0].BulkData.Unlock();
				Texture->UpdateResource();
			}
		}
	}
	return Texture;
}




```