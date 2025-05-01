# Online Subsystem

This contains notes about the online subsystem

---
## Configure the project for Steam
- In the `DefaultGame.ini` file, add the following lines of ini code:

```ini
[/Script/Engine.GameEngine]
+NetDriverDefinitions=(DefName="GameNetDriver",DriverClassName="OnlineSubsystemSteam.SteamNetDriver",DriverClassNameFallback="OnlineSubsystemUtils.IpNetDriver")

[OnlineSubsystem]
DefaultPlatformService=Steam

[OnlineSubsystemSteam]
bEnabled=true
SteamDevAppId=480

; If using Sessions
bInitServerOnClient=true

[/Script/OnlineSubsystemSteam.SteamNetDriver]
NetConnectionClassName="OnlineSubsystemSteam.SteamNetConnection"
```

- Enable the `Online Subsystem Steam` plugin in the editor
- Add `"OnlineSubsystemSteam"` and `"OnlineSubsystem"` to the .build file

---
## Accessing the subsystem
We use sessions to handle our multiplayer connections, and the `OnlineSubsystem` comes with an interface, `IOnlineSessionInterface`. We can store a pointer to that interface like so:

```cpp
// Pointer to the online session interface
TSharedPtr<class IOnlineSession, ESPMode::ThreadSafe> OnlineSessionInterface;
```

However, there is a typedef that was created for our convenience called `IOnlineSessionPtr` that we can use instead of the `TSharedPtr`:

```cpp
// Pointer to the online session interface
IOnlineSessionPtr OnlineSessionInterface;
```

>This is possible if we `#include "Interfaces/OnlineSessionInterface.h"` in the header file of the class that's storing the pointer.

Then, we can initialize and access the `OnlineSessionInterface` in the `.cpp` file like so:

```cpp
// For example purposes this is in the constructor of the player character
IOnlineSubsystem* OnlineSubsystem = IOnlineSubsystem::Get();
	if (OnlineSubsystem)
	{
		OnlineSessionInterface = OnlineSubsystem->GetSessionInterface();

		if (GEngine)
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				10,
				FColor::Cyan,
				FString::Printf(TEXT("Found Subsystem %s"),
				*OnlineSubsystem->GetSubsystemName().ToString())
			);
		}
    }    
```

---
## Creating a Session
Steam creates sessions for us, which takes time since data has to travel across the internet, so the `IOnlineSessionInterface` has delegates we can create and bind callbacks to that provide information to us when Steam sends data back to Unreal.

First, we need a function we can call that will handle creating a session for us:

```cpp
UFUNCTION(BlueprintCallable)
	void CreateGameSession();
```
>Note that this is NOT called `CreateSession()` as that function is the blueprint version of what we're about to encapsulate in `CreateGameSession()`

Next we need to create a member variable of type `FOnCreateSessionCompleteDelegate`:

```cpp
FOnCreateSessionCompleteDelegate CreateSessionCompleteDelegate;
```

Before we initialize the delegate, we need to create a callback function which has a specific signature:

```cpp
void OnCreateSessionComplete_Callback(FName SessionName, bool bWasSuccessful);
```

Now can initialize the delegate in the constructor and bind our callback at the same time like so:

```cpp
AMenuSystem_MPCharacter::AMenuSystem_MPCharacter():
	CreateSessionCompleteDelegate(FOnCreateSessionCompleteDelegate::CreateUObject(this, &ThisClass::OnCreateSessionComplete_Callback))
{
	// other constructor code
}
```

Now that our delegate is setup and the callback is bound to it, we can actually create a session:

```cpp
void AMenuSystem_MPCharacter::CreateGameSession()
{
	// If the session interface is null, return
	if (!OnlineSessionInterface.IsValid()) return;

	// Check if a session already exists before creating a new one, and if it does, destroy it.
	FNamedOnlineSession* ExistingSession = OnlineSessionInterface->GetNamedSession(NAME_GameSession);
	if (ExistingSession != nullptr)
	{
		OnlineSessionInterface->DestroySession(NAME_GameSession);
	}

	// Make sure the session interface knows which delegate to fire when Steam successfully creates a session
	OnlineSessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegate);
	
	// Configure the session settings and create the session
	TSharedPtr<FOnlineSessionSettings> SessionSettings = MakeShareable(new FOnlineSessionSettings());
	SessionSettings->bIsLANMatch = false;
	SessionSettings->NumPublicConnections = 4;
	SessionSettings->bAllowJoinInProgress = true;
	SessionSettings->bAllowJoinViaPresence = true;
	SessionSettings->bShouldAdvertise = true;
	SessionSettings->bUsesPresence = true;
	SessionSettings->bUseLobbiesIfAvailable = true;
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	OnlineSessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *SessionSettings);
}
```