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


# UClass()的修饰符
 1. Config="ConfigFileName" ， FileName为配置文件名,最后的文件名为"ConfigFileName.ini" 

    ```
     e.i
     [/ConfigTest.ConfigTestActor]     //[PathName.ClassName]
        string = "ConfigTestActor"
        BP_Set = "NewBlueprint_C"
    ``` 
    ----

    若配置文件名称为内置名，则
       ```
       e.i
     [/Script/ConfigTest.ConfigTestActor]  //[/Script/PathName.ClassName]
        string = "cdasgav"

    ```

2. PerObjectConfig  <span style="color:red"> Not working on PIE</span>

    为每个instance做配置
    ```
    e.i.
    [Vertex:PersistentLevel.NewBlueprint_C_3 Newblueprint_C]   //[ObjectName ClassName]
    string = "NewBPPPP34"
    ```

#UPROPERTY()的修饰符
    1. Config
        标识属性写入配置文件

    2.GlobalConfig
        标识属性写入配置文件
        子类配置不可修改