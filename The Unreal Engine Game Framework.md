# relative header file: LaunchEngineLoop.h/cpp

UnrealEngineModule:
		Core,
		CoreUObject,
		InputCore,
		Projects
		Engine
		SlateCore,
		Slate
		SlateRHIRender,
		UMG,
		UMGEditor

Engine Phase: Lanuch -> Preinit

# preinit:
	module loading:
	LoadCoreModules - CoreUObject
	LoadPreInitModules - Endine,Renderer...
	Project&Plugin - EarliestPossible,PostConfigInit,PostSplashScreen,PreEarlyLoadingScreen
	LoadStartupCoreModules
	Project & Plugin - PreLoadingScreen,PreDefault,Default,PostDefault
	

ModuleLoaded:
	Register UObject classes defined in module
	 constructs CDO(Class Prototype)

# Init: 

1. Load UGameEngine class specified in Engine config file; 
2. Create new UGameEngine and enshrine it as the global UEngine object;
3. Prase Commandline; 
   
   ->GEngine::ParseCommandline()
4. Init Timing Option from Commandline; 
   
   ->FEngineLoop::InitTime()
5. <font color='yellow'>Engine Init;</font> 
   
   ->GEngine->Init(IEngineLoop*)
6. <font color='green'>OnPostEngineInit callback; </font> 

	->FCoreDelegates::OnPostEngineInit.Broadcast();
   
7. EngineService startup if support multithreading;
8. Load all the post-engine init modules; 
   
    ->IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PostEngineInit) 

	->IPluginManager::Get().LoadModulesForEnabledPlugins(ELoadingPhase::PostEngineInit)
9.  <font color='green'>Call module loading phases completion callbacks; </font>  
    
	->FCoreDelegates::OnAllModuleLoadingPhasesComplete.Broadcast();

10. <font color='yellow'>Engine Start; </font>
    
	UEngine::Start()

11. initialize media framework if is client;
12. Set GIsRunning = true;
13. if not Editor, globally enable rendering;

	->FViewport::SetGameRenderingEnabled(true, 3);

14. @todo bind game loop Starved delegate; 
    
	->FCoreDelegates::StarvedGameLoop.BindStatic(&GameLoopIsStarved);
	
15. Ready to measure thread heartbeat; 

	->FThreadHeartBeat::Get().Start();

16. pause shader precomplation; 
    
    ->FShaderPipelineCache::PauseBatching();

17. <font color='green'>call OnFEngineLoopInitComplete;</font> 
    
	->FCoreDelegates::OnFEngineLoopInitComplete.Broadcast()

18. resume shader precomplation;
19. <font color='green'>call all delay delegate</font> 
    
	->FDelayedAutoRegisterHelper::RunAndClearDelayedAutoRegisterDelegates(EDelayedRegisterRunPhase::EndOfEngineInit); 


--- 



LoadMap:

	PreLoadMap - Let any interested parties know that the current world is about to be unloaded
	Notify the GameInstance of the map change
	Load UWorld package
	Give the world a reference to the gameInstance - GWorld = NewWorld;
	Get world ready for gameplay
	Load any per-map packages and make sure "always loaded" sub-level fully loaded

	InitializeActorsForPlay:
		Register all actor components in persistent level only:
			PostRegister
			Regist

			if(Primitive) {add to world->Scene (RenderingThread of UWorld)}

		Initialize the game mode:spawn AGameSession

		Route InitializeComponents(and PreInit/PostInit) to all actor,level by level:
			PreInitializeComponent for each actors

			InitializeComponent and PostInitializeComponents for each Actors

	Spawn Player:ULocalPlayer::SpawnPlayActor()

	RestartPlayer
-------
 GameInstance:
  spun off from UEngine to handle specific project funtion


  FWorldContext: track which world is loaded up at the moment

  .umap:a package serialized Levels and Actors
----
  AGamemodeBase:
   PreInitializeComponent:
   	Spawn GameState Actor
   	Associate GameState with world
   	If(Server) Spawn GameNetworkManager
   	InitGameState

   -----
   AActor InitializeComponets





---
# ENGINE LOOP TICK
	main function: FEngineLoop::Tick()

1. check if need exit; 
   
   ->	BeginExitIfRequested();

2. Send a heartbeat for the diagnostics thread; 

	-> FThreadHeartBeat::Get().HeartBeat(true);

3. Tick hotfixable values in config;
4. block movie player;
5. do profiller stuff
6. execute callbacks for cvar changes; 
   
   -> IConsoleManager::Get().CallAllConsoleVariableSinks();

7. <font color='green'>call delegate for begin frame;</font> 
   
   ->FCoreDelegates::OnBeginFrame.Broadcast();

8. flush debug output which has been buffered by other threads 

	-> GLog->FlushThreadedLogs(EOutputDeviceRedirectorFlushOptions::Async);

9. exit if frame limit is reached in benchmark mode, or if time limit is reached;
10. set FApp::CurrentTime, FApp::DeltaTime and potentially wait to enforce max tick rate;
11. beginning of RHI frame; 
    
	```cpp
	ENQUEUE_RENDER_COMMAND(BeginFrame)([CurrentFrameCounter](FRHICommandListImmediate& RHICmdList)
	{
		BeginFrameRenderThread(RHICmdList, CurrentFrameCounter);
	});

	```
12. @todo maybe call for all scene to render;
    
	```cpp
		for (FSceneInterface* Scene : GetRendererModule().GetAllocatedScenes())
	{
		ENQUEUE_RENDER_COMMAND(FScene_StartFrame)([Scene](FRHICommandListImmediate& RHICmdList)
		{
			Scene->StartFrame();
		});
	}
	```
13. if in client, When not in editor, we emit dynamic resolution's begin frame right after RHI's.
14. if in client, tick performance monitoring;
15. update memory allocator stats;
16. advance stat for render thread;

	->	FStats::AdvanceFrame( false, FStats::FOnAdvanceRenderingThreadStats::CreateStatic( &AdvanceRenderingThreadStatsGT ) );

17. Calculates average FPS/MS (outside STATS on purpose)
18. handle some per-frame tasks on the rendering thread
    ```cpp
			ENQUEUE_RENDER_COMMAND(ResetDeferredUpdates)(
			[](FRHICommandList& RHICmdList)
			{
				FDeferredUpdateResource::ResetNeedsUpdate();
				FlushPendingDeleteRHIResources_RenderThread();
			});
	```

19.  pump messages if no embedded

	-> 	FPlatformApplicationMisc::PumpMessages(true);

20. yield cpu time if engine is idle mode;
21. set WorldToMetersScale from GWorld or PlayWorld;
22. @todo tick active platform files;
    
	-> 	FPlatformFileManager::Get().TickActivePlatformFile();
23. <font color='green'>Roughly track the time when the input was sampled</font> 
    
	-> FCoreDelegates::OnSamplingInput.Broadcast();

24. process accumulated Slate input;
    
    FSlateApplication::PollGameDeviceState(); 

	FSlateApplication::FinishedInputThisFrame(); // Gives widgets a chance to process any accumulated input

25. init for RDG resource dump; 
    
	FRDGBuilder::InitResourceDump();

26. <font color='yellow'>main game engine tick</font>

	GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);

27. If a movie that is blocking the game thread has been playing,
 wait for it to finish before we continue to tick or tick again
 We do this right after GEngine->Tick() because that is where user code would initiate a load / movie.

    1.  if is client, if is ok to start up the movie player, Enable the MoviePlayer now that the preload screen manager is done
    2.  Destroy / Clean Up PreLoadScreenManager as we are now done;
        
    	-> FPreLoadScreenManager::Destroy();

28. fetches completed tasks and applies them to the scene. 
    
	->FAssetCompilingManager::Get().ProcessAsyncTasks(true)

29. end movie player block;
30. @todo Tick the platform and input portion of Slate application;

	ProcessLocalPlayerSlateOperations() 

	FSlateApplication::Get().Tick(ESlateTickType::PlatformAndInput)

31. process concurrent Slate tasks
32. @todo Tick(Advance) Time for the application and then tick and paint slate application widgets

	FSlateApplication::Get().Tick(bRenderingSuspended ? ESlateTickType::Time : ESlateTickType::TimeAndWidgets);

33. @todo pause concurrent task, WaitForOutstandingTasksOnly_for_DelaySceneRenderCompletion 
    
	```cpp
		
		if (ConcurrentTask.GetReference())
		{
			CSV_SCOPED_SET_WAIT_STAT(ConcurrentWithSlate);

			QUICK_SCOPE_CYCLE_COUNTER(STAT_ConcurrentWithSlateTickTasks_Wait);
			check(ConcurrentTaskCompleteEvent);
			ConcurrentTaskCompleteEvent->Wait();
			FPlatformProcess::ReturnSynchEventToPool(ConcurrentTaskCompleteEvent);
			ConcurrentTaskCompleteEvent = nullptr;
			ConcurrentTask = nullptr;
		}
		{
			ENQUEUE_RENDER_COMMAND(WaitForOutstandingTasksOnly_for_DelaySceneRenderCompletion)(
				[](FRHICommandList& RHICmdList)
				{
					QUICK_SCOPE_CYCLE_COUNTER(STAT_DelaySceneRenderCompletion_TaskWait);
					FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(EImmediateFlushType::WaitForOutstandingTasksOnly);
				});
		}
	```
34. clear stats
35. notify render thread end frame;
    
	```cpp
	for (FSceneInterface* Scene : GetRendererModule().GetAllocatedScenes())
	{
		ENQUEUE_RENDER_COMMAND(FScene_EndFrame)([Scene](FRHICommandListImmediate& RHICmdList)
		{
			Scene->EndFrame(RHICmdList);
		});
	}
	```
36. tick render hardware interface,update RHI
    
	FEngineLoop::RHITick( FApp::GetDeltaTime() )

37. set Simulation Latency Marker End before EndFrameRenderThread is enqueued.
    
	GEngine->SetSimulationLatencyMarkerEnd(CurrentFrameCounter);

38. Increment global frame counter. Once for each engine tick

	```cpp
	GFrameCounter++;

	ENQUEUE_RENDER_COMMAND(FrameCounter)(
	[CurrentFrameCounter = GFrameCounter](FRHICommandListImmediate& RHICmdList)
	{
		GFrameCounterRenderThread = CurrentFrameCounter;
	});
	```
39. Disregard first few ticks(<font color='yellow'>6 ticks</font>) for total tick time as it includes loading and such.
40. Find the objects which need to be cleaned up the next frame
41. Sync game and render thread. Either total sync or allowing one frame lag
    
	FFrameEndSync::Sync( CVarAllowOneFrameThreadLag->GetValueOnGameThread() != 0 );
42. tick core ticker, threads & deferred commands
    1.  Delete the objects which were enqueued for deferred cleanup before the previous frame.
    2.  destroy all linkers pending delete
    3.  do stuff 
    ```cpp
	FTSTicker::GetCoreTicker().Tick(FApp::GetDeltaTime());
	FThreadManager::Get().Tick();
	GEngine->TickDeferredCommands();
	```
43. make sure movie player not running
44. tick media framework
    
	MediaModule->TickPostRender();

45. end frame delegate;
    
	FCoreDelegates::OnEndFrame.Broadcast();

46. end of RDG resource dump
47. @todo emit dynamic resolution's end frame right before RHI's. GEngine is going to ignore it if no BeginFrame was done
48. end RHI frame
    ```cpp
	ENQUEUE_RENDER_COMMAND(EndFrame)(
	[CurrentFrameCounter](FRHICommandListImmediate& RHICmdList)
	{
		EndFrameRenderThread(RHICmdList, CurrentFrameCounter);
	});
	```
49. Set CPU utilization stats
50. @todo Set the UObject count stat

---

# Initialize engine (UEngine)
	main file: UnrealEngine.cpp
	Create UGameInstance and UGameViewprotClient and aULocalPlayer
	Initialize late-loaded modules
	start game:load default map

	main function: UEngine::Init

1. if EDITOR
   1. @todo add crash reporter for all enable plugin,if Engine is not internal Build
2. Set the memory warning handler
3. save EngineLoop instance
4. RegisterEngineElements
5. init Engine Subsystems
   ```cpp
   	FURL::StaticInit();
	EngineSubsystemCollection.Initialize(this);
   ```
6. if not SHIPPING
   1. Check for overrides to the default map on the command line. <font color='#EEB422'>DEFAULTMAP=xxx</font>
7. initial value at 100 FPS so fast machines don't have to creep up to a good frame rate due to code limiting upward "mobility" 
   
   InitializeRunningAverageDeltaTime()

8. add UEngine to root
9. <font color='green'>bind PreGarbageCollectDelegate</font>
    
	FCoreUObjectDelegates::GetPreGarbageCollectDelegate().AddStatic(UEngine::PreGarbageCollect)

10. if project name not empty
    1.  Initialize the HMDs and motion controllers, if any
    2.  Initialize attached eye tracking devices, if any
11. Disable the screensaver when running the game
12. if not Server mode && not Cmmandlet mode
    1.  If Slate is being used, initialize the renderer after RHIInit 
    2.  Slate app init sound
13. Assign thumbnail compressor/decompressor
14. @todo load Uengine object
15. reads the Engine.ini file to get the proper DefaultMaterial, etc.
16. set process limits as configured in Engine.ini or elsewhere
17. set default selected material color,Selection Outline Color
18. load WorldPartitionEditor module,if Editor
19. Initialize Object References
    
	InitializeObjectReferences()

20. enable bEnableOnScreenDebugMessages from GEngineIni;
21. Update Script Maximum loop iteration count
22. set global near clip plane
23. init Text render component's MID cache

	UTextRenderComponent::InitializeMIDCache()

24. Initialize scene cached cvars
25. Create a WorldContext for the editor to use and create an initially empty world.
26. Make sure networking checksum has access to project version
    ```cpp
		const UGeneralProjectSettings& ProjectSettings = *GetDefault<UGeneralProjectSettings>();
		FNetworkVersion::SetProjectVersion(*ProjectSettings.ProjectVersion);
	```

27. if not SHIPPING or ENABLE_PGO_PROFILE
    1.  Optionally Exec an exec file
    2.  Optionally exec commands passed in the command line.
    3.  optionally set the vsync console variable
    4.  optionally set the vsync console variable
28. notify boot process finish, and can write boot cache, and get rid of boot cache

	```cpp
		if (GetDerivedDataCache())
		{
			GetDerivedDataCacheRef().NotifyBootComplete();
		}
	```

29. register the engine with the travel and network failure broadcasts.games can override these to provide proper behavior in each error case
    ```cpp
	OnTravelFailure().AddUObject(this, &UEngine::HandleTravelFailure);
	OnNetworkFailure().AddUObject(this, &UEngine::HandleNetworkFailure);
	OnNetworkLagStateChanged().AddUObject(this, &UEngine::HandleNetworkLagStateChanged);
	```

30. IsTextureStreamingEnabled log
31. <font color='green'>Initialize the online subsystem</font>

	```cpp
	FOnlineExternalUIChanged OnExternalUIChangeDelegate;
	OnExternalUIChangeDelegate.BindUObject(this, &UEngine::OnExternalUIChange);
	UOnlineEngineInterface::Get()->BindToExternalUIOpening(OnExternalUIChangeDelegate);
	```
32. Initialise buffer visualization system data
    ```cpp
	GetBufferVisualizationData().Initialize();
	GetNaniteVisualizationData().Initialize();
	GetVirtualShadowMapVisualizationData().Initialize();
	```
33. @todo Initialize Portal services
34. Connect the engine analytics provider

	FEngineAnalytics::Initialize()

35.  Initialize the audio device after a world context is setup
36.  Dynamically load engine runtime modules
    ```cpp
		{
		FModuleManager::Get().LoadModule("ImageWriteQueue");
		FModuleManager::Get().LoadModuleChecked("StreamingPauseRendering");
		FModuleManager::Get().LoadModuleChecked("MovieScene");
		FModuleManager::Get().LoadModuleChecked("MovieSceneTracks");
		FModuleManager::Get().LoadModule("LevelSequence");
		#if WITH_EDITOR
			// The SparseVolumeTexture module containing the importer is only loaded and used in the editor.
			FModuleManager::Get().LoadModuleChecked("SparseVolumeTexture");
		#endif
	}
	```
37. Record large world coordinate state
38. Finish asset manager loading
39. @todo set bIsRHS
40. Add the stats to the EngineStats list, note this is also the order that they get rendered in if active
41. <font color='green'>Let any listeners know about the new stats</font>

	NewStatDelegate

42. report extra development memory if not SHIPPING

43. InitThreadConfig

## InitEngine (UGameEngine: UEngine)
1. call base Init()
2. Load and apply user game settings
3. <font color='yellow'>create game instance</font>

	GameInstance->InitializeStandalone()

4. if with editor
   1. init movie scene capture

5. <font color='yellow'>init viewport client</font>

	ViewportClient->Init(*GameInstance->GetWorldContext(), GameInstance);

6. Attach the viewport client to a new viewport.
    1. load config and screate game window

		GameViewportWindow = CreateGameWindow();
		
	2. <font color='yellow'>Create game viewport</font> 
       1. Creates the viewport widget where the games Slate UI is added to

			CreateGameViewportWidget( GameViewportClient );

		2. create primary scene viewport
			```cpp
			SceneViewport = MakeShareable( GameViewportClient->CreateGameViewport(GameViewportWidgetRef) );
			GameViewportClient->Viewport = SceneViewport.Get();
			
			```
		3. set viewport interface to viewport widget
		4. @todo set viewport frame
		5. @todo set sence viewport to game layer manager
    3. Changes the game window to use the game viewport instead of any loading screen or movie that might be using it instead

		SwitchGameWindowToUseGameViewport()
	
	4. <font color='yellow'>Setup Initital local player</font>
      	1. Create the viewport's console.
      	2. Create the initial player
   
			ViewportGameInstance->CreateInitialPlayer(OutError);

	5. set bIsInitialized = true for IsInitialized()
	

# Engine Start(UGameEngine: UnrealEngine)
	UGameEngine::Start()

1. <font color='yellow'> GameInstance call start game instance to start game.</font>
```cpp
GameInstance->StartGameInstance();
```

# Start Game 
	void UGameInstance::StartGameInstance() 

1. load game ini create default URL map. <font color='#EEB422'>[DefaultPlayer]</font>
2. override map if not SHIPPING build.Or allow SHIPPING build on server ,or running PGO profiling
3. Get GameMapsSettings CDO to get DefaultMap.
4. get override map package name
5. <font color='yellow'>Browse URL to connect to a server map or Load Map in local play.</font>

	```cpp
	EBrowseReturnVal::Type UEngine::Browse( FWorldContext& WorldContext, FURL URL, FString& Error )
	{
	...
	//if run Locally just load map
	// Local map file.
	return  UEngine::LoadMap( WorldContext, URL, NULL, Error ) ? EBrowseReturnVal::Success : EBrowseReturnVal::Failure;
	...
	}
	```
6. <font color='green'>broadcasts on start.</font>

	```cpp
	FWorldDelegates::OnStartGameInstance
	```

7. call UGameInstance::OnStart()


# UEngine::Load Map()
1. Map URL remove PIE prefix
2. make sure level streaming isn't frozen

```cpp
if (WorldContext.World())
{
	WorldContext.World()->bIsLevelStreamingFrozen = false;
}
```

3. <font color='green'>preload delegate msg broadcasts</font> and stop moive player tick
```cpp
FCoreUObjectDelegates::PreLoadMapWithContext.Broadcast(WorldContext, URL.Map);
FCoreUObjectDelegates::PreLoadMap.Broadcast(URL.Map);
FMoviePlayerProxy::BlockingTick();
```

4. make sure there is a matching PostLoadMap() no matter how we exit

```cpp
struct FPostLoadMapCaller
{
	FPostLoadMapCaller()
		: bCalled(false)
	{}

	~FPostLoadMapCaller()
	{
		if (!bCalled)
		{
			FCoreUObjectDelegates::PostLoadMapWithWorld.Broadcast(nullptr);
		}
	}

	void Broadcast(UWorld* World)
	{
		if (ensure(!bCalled))
		{
			bCalled = true;
			FCoreUObjectDelegates::PostLoadMapWithWorld.Broadcast(World);
		}
	}

private:
	bool bCalled;

} PostLoadMapCaller;
```
5. cancel oustanding Texture Streaming works

```cpp
UTexture2D::CancelPendingTextureStreaming();
```
6. play a load map movie if specified in ini
7. clean up any per-map loaded packages for the map we are leaving
8. cleanup the existing per-game pacakges
9. <font color='yellow'>Cancel any pending async map changes after flushing async loading. We flush async loading before canceling the map change to avoid completion after cancellation to not leave references to the "to be changed to" level around. Async loading is implicitly flushed again later on during garbage collection</font>

```cpp
FlushAsyncLoading();
CancelPendingMapChange(WorldContext);
WorldContext.SeamlessTravelHandler.CancelTravel();

```

10. init blueprint context Tracker

```cpp
GInitRunaway()
```

11. unload current world
    1.  <font color='green'>Notify listeners that all levels will be removed before we start tearing down the world</font>
    ```cpp
	FWorldDelegates::PreLevelRemovedFromWorld.Broadcast(nullptr, WorldContext.World())
	```
    2. <font color='green'>mark world as being torn down</font>
    ```cpp
		void UWorld::BeginTearingDown()
	{
		bIsTearingDown = true;
		UE_LOG(LogWorld, Log, TEXT("BeginTearingDown for %s"), *GetOutermost()->GetName());

		//Simultaneous similar edits that caused merge conflict. Taking both for now to unblock.
		//Can likely be unified.
		FWorldDelegates::OnWorldBeginTearDown.Broadcast(this);
	}
	```
    3. if URL has no quiet option
       1. set current map transition type to ETransitionType::Loading.
       2. set transition game-mode if is set.<font color='#EEB422'>「Game=xxx」</font>
       3. Display loading screen.
       4. <font color='green'> Check if a loading movie is playing.  If so it is not safe to redraw the viewport due to potential race conditions with font rendering</font>
		```cpp
		bool bIsDesktopLoadingMovieCurrentlyPlaying = FCoreDelegates::IsLoadingMovieCurrentlyPlaying.IsBound() ? FCoreDelegates::IsLoadingMovieCurrentlyPlaying.Execute() : false;
		```
       5. Check if a loading movie is playing on an XR device as well
       6. call LoadMapRedrawViewports() if no loading movie playing
       7. set current map transition type to ETransitionType::None
    4. Clean up networking
    5. @todo Make sure there are no pending visibility requests for levels
		```cpp
		WorldContext.World()->FlushLevelStreaming(EFlushLevelStreamingType::Visibility);
		```
    6. <font color='green'>send a message that all levels are going away (NULL means every sublevel is being removed without a call to RemoveFromWorld for each)</font>
		```cpp
		FWorldDelegates::LevelRemovedFromWorld.Broadcast(nullptr, WorldContext.World())
		```
    7. Disassociate the players from their PlayerControllers in this world.
    8. Route EndPlay for each actor
    9. clean up world
    10. clear any "DISPLAY" properties referencing level objects
    11. Remove world object from root
    12. mark everything else contained in the world to be deleted
    13. If an unloaded levelstreaming still has a loaded level we need to mark its objects to be deleted as well
    14. Stop all audio to remove references to current level
12. rim memory to clear up allocations from the previous level (also flushes rendering)
    ```cpp
	if (GDelayTrimMemoryDuringMapLoadMode == 0)
	{
		TrimMemory();
	}
	```
13. Cancels the Forced StreamType for textures using a timer.
14. @todo DefragmentTexturePool if require cooked data.
15.  UGameInstance::PreloadContentForURL
16.  If this world is a PIE instance, we need to check if we are traveling to another PIE instance's world.If we are, we need to set the PIERemapPrefix so that we load a copy of that world, instead of loading the PIE world directly.
17. normal map loading
    1.  Set the world type in the static map, so that UWorld::PostLoad can set the world type
    2. @todo See if the level is already in memory
    3. If the level isn't already in memory, load level from disk
    4. Clean up the world type list now that PostLoad has occurred
    5. Find the newly loaded world.
    6. If the world was not found, it could be a redirector to a world. If so, follow it to the destination world.
    7. create map build data regisitry entry
		```cpp
		NewWorld->PersistentLevel->HandleLegacyMapBuildData();
		```
    8. if world type is PIE, duplicate world for NewWorld,or rename world name to PIE world format.
       1. use same FeatureLevel as editor.if WITH_EDITOR.
    9. else if world type is GAME. Create PIE world load from package.
18. set new world to game instance.
19. set GWorld = NewWorld.
20. sync WorldContext's Current world and world type.
21. <font color='yellow'>init new world</font>
    1.  case PIE world:
    	PIE worlds are not added to root and are initialized differently
		```cpp
		PostCreatePIEWorld(WorldContext.World())
		```
    2.  else:
		add world to root. init world.
		```cpp
		WorldContext.World()->InitWorld();
		```
22. Handle pending level.
23. <font color='yellow'>spawn game-mode</font>
    ```cpp
	WorldContext.World()->SetGameMode(URL)
	```
24. initialize default base soundmix
25. Listen for clients.if is server.
26. @todo load mutator pacakges.<font color='#EEB422'>「Mutator=xxx」</font>
27. Process global shader results before we try to render anything

	```cpp
	if (GShaderCompilingManager)
	{
		GShaderCompilingManager->ProcessAsyncResults(false, true);
	}
	```

28. load any per-map packages
29. Make sure "always loaded" sub-levels are fully loaded
    ```cpp
	WorldContext.World()->FlushLevelStreaming(EFlushLevelStreamingType::Visibility);
	```
30. Gives a chance to any assets being used for PIE/game to complete
    ```cpp
	FAssetCompilingManager::Get().ProcessAsyncTasks();
	```
31. Create AI System if needed.

	```cpp
	WorldContext.World()->CreateAISystem();
	```

32. <font color='yellow'>Initialize gameplay for the level</font>
    1.  init actor for play
    2.  parallel process component register

33. add navigation system
34. Spawn play actors for all active local players
35. <font color='green'>Prime texture streaming.</font>
```cpp
IStreamingManager::Get().NotifyLevelChange()
```
36. XRSystem begin play
37. <font color='green'>post load map delegate</font>
```cpp
PostLoadMapCaller.Broadcast(WorldContext.World());
```
38. @todo set for one tick after completely loading and initializing a new world
39. update streaming immediately so that there's no tick prior to processing any levels that should be initially visible that requires calculating the scene

```cpp
RedrawViewports(false)
```
40. RedrawViewports() may have added a dummy playerstart location. Remove all views to start from fresh the next Tick()
    
```cpp
IStreamingManager::Get().RemoveStreamingViews( RemoveStreamingViews_All )
```

41. See if we need to record network demos. <font color='#EEB422'>「DemoRec=xxx」</font>
42. GameInstance::LoadComplete()
43. perform the delayed TrimMemory if desired

---

# UGameEngine::Tick
1. flush log
2. clean up game viewport
3. tick media module.

```cpp
MediaModule->TickPreEngine()
```
4. tick analytics
5. begin ticking worlds 
   1. switch GWorld to World that going to tick
   2. Tick all travel and Pending NetGames (Seamless, server, client)
   3. <font color='yellow'>tick world if not idlemdoe.</font>
   ```cpp
   Context.World()->Tick( LEVELTICK_All, DeltaSeconds )
   ```
   4. Only update reflection captures in game once all 'always loaded' levels have been loaded
   
---

# UWorld::InitWorld
1. <font color='green'>bind on post GC</font>

```cpp
FCoreUObjectDelegates::GetPostGarbageCollect().AddUObject(this, &UWorld::OnPostGC)
```
2. <font color='yellow'>subsystem init</font>

3. <font color='green'>on preworldInitialization broadcast</font>

```cpp
FWorldDelegates::OnPreWorldInitialization.Broadcast(this, IVS)
```
4. create physics scene if need
5. Prepare AI systems
6. Material Parameter Collection Instance Setup
7. set current level as PersistentLevel.then add to Levels.
8. If we are not Seamless Traveling remove PersistentLevel from LevelCollection if it is in a collection
9. initialize DefaultPhysicsVolume for the world
10. Find gravity
11. Create physics collision handler, if we have a physics scene
12. Conditionally Create Default Level Collections.Dynamic & Static Level Collection will be create.
13. mark World initialized.
14. <font color='green'>call general post initialization delegates</font>

```cpp
FWorldDelegates::OnPostWorldInitialization.Broadcast(this, IVS)
```

15. Precomputed Visibility Handler & VolumeDistanceField.
16.  Initialize Rendering Resources.add precomputed light volume & lightmap to scene, ini build data rendering resource.
17. <font color='yellow'>do PersistentLevel on loaded  WorldPartition stuff</font> 
```cpp
PersistentLevel->OnLevelLoaded()
```
1.  add PersistentLevel to streaming manager.
2.  finialize all world subsystem, call world subsystem's PostInit()

```cpp
PostInitializeSubsystems()
```
20. BroadcastLevelsChanged().
    ```cpp
	void UWorld::BroadcastLevelsChanged()
	{
		LevelsChangedEvent.Broadcast();
	#if WITH_EDITOR
		FWorldDelegates::RefreshLevelScriptActions.Broadcast(this);
	#endif	//#if WITH_EDITOR
	}
	```
21. Asset Tags Finalized

---
