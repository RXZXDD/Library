FMeshBatch

FDrawCommand

FMeshPassProcessor build 1+ FMeshDrawCommands From a FMeshBatch

Unufrom Buffers
- PrimitiveUniformBuffer
- MaterialUniformBuffer
- MaterialParameterCollections
- PrecomputedLightingUnifromBuffer
- PassUniformBuffers

Cache Invalidation
debug use
    ```c++

    VALITDATE_UNIFORM_BUFFER_LIFETIME

    ```


Vertext Factories
    generate vertex data


## High level frame with caching
FprimitiveSceneInfo::AddToScene

    - if static,cache FMeshBatch's on FPrimtitiveSceneInfo
        - Also generate FMeshDrawCommand and store on the scene

FScene:: SetSkyLight
    -Invalidate cached FMeshDrawCommands

InitViews
    - foreach Primitive
      - if Static Relevance
        - Compute LOD and add cached FmeshDrawCommands to visible List
      - if Dynamic Relevance
        - Gather FMeshBatch's with GetDynamicMeshElements()

RenderDepth
    - ForeachFMeshDrawCommand in visible list, Draw 

Dynamic Instancing
    Can robustly merge by compareing FMeshDrawCommands

    - demelite
      - Assigning state buckets from merging is slow
      - cannot addord to compare bindings each frame
      - cache at FPrimitive::AddToScene 

Draw Call Merging
    Actual merging operation is just a data transformation on the visible list
    Any commands in the same state bucket get replaced with a single instanced FMeshDrawCommand

# Reffer
[FestEurope2019]UE4网格渲染管线重构
    https://www.bilibili.com/video/BV1gJ411J7j1/?vd_source=8f4cdd220db3fddd745de6bc17511b9e