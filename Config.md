# Config

## Add Custom Setting Item and file
- Ways to do
    1. <b>inherit UDeveloperSettings</b>
    ```c++
    //UAssetManagerSettings.h
    UCLASS(config = Game, defaultconfig, ...)
    class ENGINE_API UAssetManagerSettings : public UDeveloperSettings
    {
        ...
        /** List of asset types to scan at startup */
        UPROPERTY(config, EditAnywhere, Category = "Asset Manager")
        TArray<FPrimaryAssetTypeInfo> PrimaryAssetTypesToScan;
        ...
    }
    ```

    2. <b>ISettingsModule</b>

        All UObject Usable, WITH_EDITOR only

        Suit for Plugin setting

        ```c++
        //GameplayTagsEditorModule.cpp
        //StartupModule()函数中
        if (ISettingsModule* SettingsModule = FModuleManager::GetModulePtr<ISettingsModule>("Settings"))
        {
            SettingsModule->RegisterSettings("Project", "Project", "GameplayTags",
                                            LOCTEXT("GameplayTagSettingsName", "GameplayTags"),
                                            LOCTEXT("GameplayTagSettingsNameDesc", "GameplayTag Settings"),
                                            GetMutableDefault<UGameplayTagsSettings>());
        }
        ...
        //ShutdownModule()函数中
        if (ISettingsModule* SettingsModule = FModuleManager::GetModulePtr<ISettingsModule>("Settings"))
        {
            SettingsModule->UnregisterSettings("Project", "Project", "GameplayTags");
        }
        ...
        ```

    3. <b>使用GConfig对象及SaveConfig、LoadConfig函数</b>

