LaunchEngineLoop.h

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

preinit:
	module loading:
	LoadCoreModules - CoreUObject
	LoadPreInitModules - Endine,Renderer...
	Project&Plugin - EarliestPossible,PostConfigInit,PostSplashScreen,PreEarlyLoadingScreen
	LoadStartupCoreModules
	Project & Plugin - PreLoadingScreen,PreDefault,Default,PostDefault
	

ModuleLoaded:
	Register UObject classes defined in module
	 constructs CDO(Class Prototype)

Init:
 Load UGameEngine class specified in Engine config file
 Create new UGameEngine and enshrine it as the global UEngine object
 Initialize engine:Create UGameInstance and UGameViewprotClient and aULocalPlayer
 Initialize late-loaded modules
 start game:load default map

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