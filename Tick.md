# Tick

## Tick Group

TG_PrePhysics - ticked before physics simulation starts

TG_StartPhysics - special tick group that starts physics simulation

TG_DuringPhysics - ticks that can be run in parallel with our physics simulation work

TG_EndPhysics - special tick group that ends physics simulation

TG_PreCloth - any item that needs physics to be complete before being executed

TG_StartCloth - any item that needs to be updated after rigid body simulation is done, but before cloth is simulation is done

TG_EndCloth - any item that can be run during cloth simulation

TG_PostPhysics - any item that needs rigid body and cloth simulation to be complete before being executed

TG_PostUpdateWork - any item that needs the update work to be done before being ticked

TG_NewlySpawned - Special tick group that is not actually a tick group. After every tick group this is repeatedly re-run until there are no more newly spawned items to run

# Actor Tick

The default tick function for an actor or component can be enabled or disabled during the game with the AActor::SetActorTickEnabled and UActorComponent::SetComponentTickEnabled functions, respectively. In addition, an actor or component can have multiple tick functions. This is accomplished by creating a struct that inherits from FTickFunction and overriding the ExecuteTick and DiagnosticMessage functions. The default actor and component tick function structures are good examples of how to build your own, and can be found in EngineBaseTypes.h under the names FActorTickFunction and FComponentTickFunction.

Once you have added your own tick function structure to your actor or component, it can be initialized, usually in the constructor of the owning class. To enable and register the tick function, the most common route is overriding AActor::RegisterActorTickFunctions and adding calls to the tick function structure's SetTickFunctionEnable, followed by RegisterTickFunction with the owning actor's level as an argument. The end result of this process is that any actor or component that you create can tick multiple times, including ticking in different groups and with individual dependencies per tick function. To set a tick dependency manually, call AddPrerequisite on the tick function structure you want to make dependent on another and pass in the tick function structure that you wish to use as your dependency.

link:https://docs.unrealengine.com/5.0/en-US/actor-ticking-in-unreal-engine/

# Add Custom Tick
1.new struct inherd from FTickFunction
```c++
truct FMyTick : public FTickFunction
{
	GENERATED_BODY()
	
	class AMyCableActor* MyTarget;
	 virtual void ExecuteTick(float DeltaTime, ELevelTick TickType, ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent) override;

		virtual FString DiagnosticMessage() override;

};
```

2. declear variable
```
	struct FMyTick MyTick;
```
3. override Actor's Tick Register Function
```
void AMyCableActor::RegisterActorTickFunctions(bool bRegister)
{
	Super::RegisterActorTickFunctions(bRegister);
	if(bRegister)
	{
		 MyTick.MyTarget = this;
		MyTick.SetTickFunctionEnable(MyTick.bStartWithTickEnabled || MyTick.IsTickFunctionEnabled());
		 MyTick.RegisterTickFunction(GetLevel());
		
	}
	else
	{
		if(MyTick.IsTickFunctionRegistered())
		{
			MyTick.UnRegisterTickFunction();			
		}
	}
	
}
```
4. add custom TickActor for call custom tick function
```
void AMyCableActor::MyTickActor(float DeltaTime, ELevelTick TickType, FMyTick& ThisTickFunction)
{
	// Non-player update.
	// If an Actor has been Destroyed or its level has been unloaded don't execute any queued ticks
	if (IsValidChecked(this) && GetWorld())
	{
		MyTickFunction(DeltaTime);	// perform any tick functions unique to an actor subclass
	}
}
```
5. add custom tick function
```
void AMyCableActor::MyTickFunction(float DeltaSeconds)
{
	UE_LOG(LogTemp, Warning, TEXT("My Tick "));
}
```