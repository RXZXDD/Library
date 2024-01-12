# RunTime Pak Usage
## Attention

<font color='red' size='10' >**Only Tried in UE Ver-5.1**</font>

## In Situation

- Want to Hot-Patch Content

 ## Process of Using Pak File at Real-Time
  1. [Preliminaries](#Preliminaries) 
   
  2. [Mount Pak](#MountPak)
   
  3. [Register Mount Point](#RegisterMountPoint)

  4. [Use Asset in Pak](#UseAssetInPak)

## Preliminaries <a id='Preliminaries'></a>
>### for the project used to use pak at runtime
>1. Uncheck setting <font color='#EEB422'>「Project -> Packaging -> IOStore」</font>
  
>### for the project you want to pacakge content into pak file：
>1. Uncheck setting <font color='#EEB422'>「Project -> Packaging -> Share Material Shader Code」</font>，for prevent shader missing after package.
>2. Use 「Primary Asset Lable」 to split content into chunk.
>3. check setting <font color='#EEB422'>「Project -> Packaging -> Generate Chunk」</font>, for chunking asset into different pak.


## Mount Pak <a id='MountPak'></a>
1. Gave a Disk Path」 of .Pak File. 
    
    Like <font color='#EEB422'>「G:\PakDemo.pak」</font>
2. Format input path to Platform File Name. 
    ```cpp
        //e.g. 「D:\\BranchTest\\a\\b.pak」 -> 「D:/BranchTest/a/b.pak」
        //api
        void FPaths::MakePlatformFilename( FString& InPath )
    ```

3. Get platform file manager of pak file

    ```cpp
    //api
    FPlatformFileManager::Get().FindPlatformFile(TEXT("PakFile"))
    ```

4. Use platform manager to mount pak file.Means that the pak file will add to PakFileManager::PakFiles as A new 「Pak Entry」.

    ```cpp
    //api
    bool FPakPlatformFile::Mount(const TCHAR* InPakFilename, uint32 PakOrder, const TCHAR* InPath /*= NULL*/, bool bLoadIndex /*= true*/)

    ```

## Register Mount Point<a id='RegisterMountPoint'></a>

1. Register Mount Point to UnrealFileSystem's Root

    ```cpp
    //api
    //RootPath: Logic Path that UFS will use it to find 「Package」while trying to Load Object
    //  Most Use RootPath: "/Game/", "/Engine/".
    //  Can Use console cmd 「PackageName.DumpMountPoints」to check
    //
    //ContentPath:Entry path where to access contain assets.In usual, can use 「IPakFile::GetMountPoint()」"
    //
    //e.g.
    //Package mount point is 「E:/UEProject/Linux2Project/Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC/」.Turns it into 「E:/UEProject/Linux2Project/Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC/」.
    //Seems that the function bellow will auto convert abs path to Project Dir relative path. For the given log: 「FPackageName: Mount point added: '../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC/' mounted to '/Game/'」
    //Or convert to Project Dir relative path manually.
    void FPackageName::RegisterMountPoint(const FString& RootPath, const FString& ContentPath)
    ```

2.  Check Mount 
   
    Use console cmd 「PackageName.DumpMountPoints」to check.

    As the example above in comment, if dumping has log contain your pak path like: 「LogPackageName: 	'/Game/' -> '../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC11/'」,it means you have sucessfully register mount point to UFS.


## Use Asset in Pak<a id='UseAssetInPak'></a>
1. Make a currect filename of the asset. 

    Format: <font color='pink'>RegisterRootPath</font>+<font color='green'>PackageName</font>.<font color='yellow'>ClassName/ObjectName</font>
    
    >e.g.    
    >- non-BP asset:<font color='pink'>/Game/</font><font color='green'>AdditionCook_DLC11/YYBMiku</font>.<font color='yellow'>YYBMiku</font>
    >
    >- BP asset:<font color='pink'>/Game/</font><font color='green'>AdditionCook_DLC/BP_DLC</font>.<font color='yellow'>BP_DLC_C</font>

    ---
    >    As example above, we register <font color='#EEB422'>「../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC11/」</font> to RootPath <font color='#EEB422'>「/Game/」</font>, 
    >
    >    you want to load asset <font color='#EEB422'>「../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook_DLC11/A/B/a.uasset」</font>, 
    >
    >    then the file format will be : <font color='pink'>/Game/</font><font color='green'>A/B/a</font>.<font color='yellow'>a</font> 
    >    
    ---
    >   Another Example:
    >   1. have a pak, in disk 「A:/b/c.pak」,and pak's mount point is <font color='#EEB422'>「../../YourProjectName/Content/」</font>.contains below uasset. 
    > 
    >       - Folder1/item1.uasset
    >       - Folder1/item2.uasset
    >       - Folder2/item3.uasset
    >       - Folder3/SubFolder/BP_item4.uasset
    >    2.  you mount the pak to PakPlatformManager, and Register the pak's mount point <font color='#EEB422'>「../../YourProjectName/Content/」</font> to UFS's Root path <font color='#EEB422'>「/MyMountRoot/」</font>
    >    3.  then you can access assets like below:
    >           |asset name in pak|load object/class file name|
    >           |:---                          |:---|
    >           |Folder1/item1.uasset           |/MyMountRoot/Folder1/item1.item1|
    >           |Folder1/item2.uasset           |/MyMountRoot/Folder1/item2.item2 |
    >           |Folder2/item3.uasset           |/MyMountRoot/Folder2/item3.item3 |
    >           |Folder3/SubFolder/item4.uasset |/MyMountRoot/Folder3/SubFolder/BP_item4.BP_item4_C    |
2. load asset.
    ```cpp
    //load an object
    template< class T > 
        inline T* LoadObject( UObject* Outer, const TCHAR* Name, const TCHAR* Filename=nullptr, uint32 LoadFlags=LOAD_None, UPackageMap* Sandbox=nullptr, const FLinkerInstancingContext* InstancingContext=nullptr )

    //load a class object
    template< class T > 
    inline UClass* LoadClass( UObject* Outer, const TCHAR* Name, const TCHAR* Filename=nullptr, uint32 LoadFlags=LOAD_None, UPackageMap* Sandbox=nullptr )

    ```
    <font color='red'>Difference</font>: 
    
    - LoadClass() will do a IsChildOf(BaseClass) to ensure the loaded object is a child of user specify Parent Class
    - In usual,load BP class use LoadClass(), other asset in content use LoadObject().


## Helpful tools.
1. UnrealPak
   
   for packaging cooked asset into different .pak file.

2. PackageName.DumpMountPoints
   
   to check current UFS mount points.

3. FPackageName class 
   
   to handle asset file name,and use to (un)register mount point.
    
