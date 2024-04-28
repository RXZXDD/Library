# RunTime Pak Usage

## Attention

**Only Tried in UE Ver-5.1**

## In Situation

* Want to Hot-Patch Content

## Process of Using Pak File at Real-Time

1. [Preliminaries](<RunTime Pak Usage.md#Preliminaries>)
2. [Mount Pak](<RunTime Pak Usage.md#MountPak>)
3. [Register Mount Point](<RunTime Pak Usage.md#RegisterMountPoint>)
4. [Use Asset in Pak](<RunTime Pak Usage.md#UseAssetInPak>)

## Preliminaries <a href="#preliminaries" id="preliminaries"></a>

> #### for the project used to use pak at runtime
>
> 1. Uncheck setting 「Project -> Packaging -> IOStore」

> #### for the project you want to pacakge content into pak file：
>
> 1. Uncheck setting 「Project -> Packaging -> Share Material Shader Code」，for prevent shader missing after package.
> 2. Use 「Primary Asset Lable」 to split content into chunk.
> 3. check setting 「Project -> Packaging -> Generate Chunk」, for chunking asset into different pak.

## Mount Pak <a href="#mountpak" id="mountpak"></a>

1.  Gave a Disk Path」 of .Pak File.

    Like 「G:\PakDemo.pak」
2.  Format input path to Platform File Name.

    ```cpp
        //e.g. 「D:\\BranchTest\\a\\b.pak」 -> 「D:/BranchTest/a/b.pak」
        //api
        void FPaths::MakePlatformFilename( FString& InPath )
    ```
3.  Get platform file manager of pak file

    ```cpp
    //api
    FPlatformFileManager::Get().FindPlatformFile(TEXT("PakFile"))
    ```
4.  Use platform manager to mount pak file.Means that the pak file will add to PakFileManager::PakFiles as A new 「Pak Entry」.

    ```cpp
    //api
    bool FPakPlatformFile::Mount(const TCHAR* InPakFilename, uint32 PakOrder, const TCHAR* InPath /*= NULL*/, bool bLoadIndex /*= true*/)

    ```

## Register Mount Point <a href="#registermountpoint" id="registermountpoint"></a>

1.  Register Mount Point to UnrealFileSystem's Root

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

    As the example above in comment, if dumping has log contain your pak path like: 「LogPackageName: '/Game/' -> '../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook\_DLC11/'」,it means you have sucessfully register mount point to UFS.

## Use Asset in Pak <a href="#useassetinpak" id="useassetinpak"></a>

1.  Make a currect filename of the asset.

    Format: RegisterRootPath+PackageName.ClassName/ObjectName

    > e.g.
    >
    > * non-BP asset:/Game/AdditionCook\_DLC11/YYBMiku.YYBMiku
    > * BP asset:/Game/AdditionCook\_DLC/BP\_DLC.BP\_DLC\_C

    ***

    > As example above, we register 「../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook\_DLC11/」 to RootPath 「/Game/」,
    >
    > you want to load asset 「../../../../../Saved/Cooked/Windows/Linux2Project/Content/AdditionCook\_DLC11/A/B/a.uasset」,
    >
    > then the file format will be : /Game/A/B/a.a

    ***

    > Another Example:
    >
    > 1. have a pak, in disk 「A:/b/c.pak」,and pak's mount point is 「../../YourProjectName/Content/」.contains below uasset.
    >    * Folder1/item1.uasset
    >    * Folder1/item2.uasset
    >    * Folder2/item3.uasset
    >    * Folder3/SubFolder/BP\_item4.uasset
    > 2. you mount the pak to PakPlatformManager, and Register the pak's mount point 「../../YourProjectName/Content/」 to UFS's Root path 「/MyMountRoot/」
    > 3.  then you can access assets like below:
    >
    >     | asset name in pak              | load object/class file name                           |
    >     | ------------------------------ | ----------------------------------------------------- |
    >     | Folder1/item1.uasset           | /MyMountRoot/Folder1/item1.item1                      |
    >     | Folder1/item2.uasset           | /MyMountRoot/Folder1/item2.item2                      |
    >     | Folder2/item3.uasset           | /MyMountRoot/Folder2/item3.item3                      |
    >     | Folder3/SubFolder/item4.uasset | /MyMountRoot/Folder3/SubFolder/BP\_item4.BP\_item4\_C |
2.  load asset.

    ```cpp
    //load an object
    template< class T > 
        inline T* LoadObject( UObject* Outer, const TCHAR* Name, const TCHAR* Filename=nullptr, uint32 LoadFlags=LOAD_None, UPackageMap* Sandbox=nullptr, const FLinkerInstancingContext* InstancingContext=nullptr )

    //load a class object
    template< class T > 
    inline UClass* LoadClass( UObject* Outer, const TCHAR* Name, const TCHAR* Filename=nullptr, uint32 LoadFlags=LOAD_None, UPackageMap* Sandbox=nullptr )

    ```

    Difference:

    * LoadClass() will do a IsChildOf(BaseClass) to ensure the loaded object is a child of user specify Parent Class
    * In usual,load BP class use LoadClass(), other asset in content use LoadObject().

## Helpful tools.

1.  UnrealPak

    for packaging cooked asset into different .pak file.
2.  PackageName.DumpMountPoints

    to check current UFS mount points.
3.  FPackageName class

    to handle asset file name,and use to (un)register mount point.
