# Config

## 使用修饰符

### UClass()的修饰符
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

### UPROPERTY()的修饰符
        1. Config
            标识属性写入配置文件

        2.GlobalConfig
            标识属性写入配置文件
            子类配置不可修改

---

## Use ISettingsModule

    适用于任何UObject
    主要使用途径：让开发者能直接从projectSetting面板查看/操作配置属性
    
 - 使用方式
   - Startup
        ```c++
        //S1---loadModule
        #if WITH_EDITOR
            ISettingsModule& SettingsModule =
                FModuleManager::LoadModuleChecked<ISettingsModule>(TEXT("Settings"));

        //S2---Register Setting
            SettingsModule.RegisterSettings(
                "Project", //Containner name
                "Project", //Category name
                "TestSection",//Section name
                FText::FromString("TestDisplayname"), 
                FText::FromString("TestDescrption"), 
                GetMutableDefault<AConfigISettingTest>() //setting object
                );
        #endif
        ```

    - shutdown
        ```c++
            #if WITH_EDITOR
            ISettingsModule& SettingsModule =
                FModuleManager::LoadModuleChecked<ISettingsModule>(TEXT("Settings"));

            SettingsModule.UnregisterSettings("Project","Project", "TestSection");

            //SettingsModule.
            #endif
        ```

---
## 直接使用GConfig

 GConfig： 引擎中唯一的配置缓存变量, 实质为TMap<"FilePath\FileName", class FConfiFile>


 1.添加新的自定义配置文件
 ```c++
    FConfigFile& ConfigFile =  GConfig->Add(path /*完整路径，e.i. E:\UEProject\PakTest\Saved\Config\WindowsEditor\GConfigTest.ini*/, 
                                            FConfigFile());
 ```

 2.读取配置文件到缓存
 ```c++
    ConfigFile.Read(path);
 ```