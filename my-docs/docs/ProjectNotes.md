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
## Sessions

Sessions are used by Steam/Unreal to connect players. Steam creates sessions for us, which takes time since data has to travel across the internet, so the `IOnlineSessionInterface` has delegates we can create and bind callbacks to that provide information to us when Steam sends data back to Unreal.

---
## Creating a Session

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

Now we can initialize the delegate in the constructor and bind our callback at the same time like so:

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
	SessionSettings->Set(FName("MatchType"), FString("FreeForAll"), EOnlineDataAdvertisementType::ViaOnlineServiceAndPing);
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	OnlineSessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *SessionSettings);
}
```
When Steam is done handling the session creation, Unreal calls `OnCreateSessionComplete_Callback()` because it was assigned to its delegate list. This allows us to call `ServerTravel()`:

```cpp
void AMenuSystem_MPCharacter::OnCreateSessionComplete_Callback(FName SessionName, bool bWasSuccessful)
{
	if (bWasSuccessful)
	{
		if (GEngine)
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				10,
				FColor::Cyan,
				FString::Printf(TEXT("Created session: %s"), *SessionName.ToString())
			);
		}

		UWorld* World = GetWorld();
		if (World)
		{
			World->ServerTravel(FString("/Game/Maps/Lobby?listen"));
		}
	}
	else
	{
		if (GEngine)
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				10,
				FColor::Red,
				FString(TEXT("Failed to create session!"))
			);
		}
	}
}
```
>We hardcoded the map name, but this can easily be a variable and `?listen` appended

This effectively moves the host to the specified level and configures it to be a listen server.

---
## Joining a Session

Just like creating a session we need a delegate, a callback function to bind to that delegate, and a function we can use to encapsulate our join logic. Joining sessions also requires that we search for and find sessions, so we need an extra delegate, callback, and a special`TSharedPtr<FOnlineSessionSearch> SessionSearch;` member variable so we can store search results:

```cpp
protected:
	UFUNCTION(BlueprintCallable)
	void JoinGameSession();

	void OnFindSessionsComplete_Callback(bool bWasSuccessful);
	void OnJoinSessionComplete_Callback(FName SessionName, EOnJoinSessionCompleteResult::Type Result);

private:
	FOnFindSessionsCompleteDelegate FindSessionsCompleteDelegate;
	FOnJoinSessionCompleteDelegate JoinSessionCompleteDelegate;

	TSharedPtr<FOnlineSessionSearch> SessionSearch; // Stores search results
```
>Make sure to initialize and bind the callback to the delegate like we did in `CreateGameSession()`

In our `JoinGameSession()` function we first add the delegate to the `OnlineSessionInterface`'s delegate list, configure our search results, and call `FindSessions()`:

```cpp
void AMenuSystem_MPCharacter::JoinGameSession()
{
	// Find game sessions

	if (!OnlineSessionInterface.IsValid()) return;

	OnlineSessionInterface->AddOnFindSessionsCompleteDelegate_Handle(FindSessionsCompleteDelegate);

	SessionSearch = MakeShareable(new FOnlineSessionSearch());
	SessionSearch->MaxSearchResults = 10000;
	SessionSearch->bIsLanQuery = false;
	SessionSearch->QuerySettings.Set(SEARCH_PRESENCE, true, EOnlineComparisonOp::Equals);

	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	OnlineSessionInterface->FindSessions(*LocalPlayer->GetPreferredUniqueNetId(), SessionSearch.ToSharedRef());
}
```
>We search for 10,000 sessions because we're using dev app ID 480 and there could be thousands of results, so our search needs to be very wide

When Steam is done handling the search, Unreal calls `OnFindSessionsComplete_Callback()` with which we can gather information about the result. There seems to be some out of date information on how to handle this next bit, as part of the Unreal code will soon be deprecated but there's no update on what to replace the deprecated bit with. Because of this, the internet has found a bit of a hacky workaround:

```cpp
void AMenuSystem_MPCharacter::OnFindSessionsComplete_Callback(bool bWasSuccessful)
{
	if (!OnlineSessionInterface.IsValid()) return;
	
	for (FOnlineSessionSearchResult Result : SessionSearch->SearchResults)
	{
		FString Id = Result.GetSessionIdStr();
		FString User = Result.Session.OwningUserName;
		FString MatchType;
		Result.Session.SessionSettings.Get(FName("MatchType"), MatchType); // "MatchType" can be changed to a variable later
		if (GEngine)
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				10,
				FColor::Yellow,
				FString::Printf(TEXT("Id: %s, User: %s"), *Id, *User)
			);
		}
		if (MatchType == "FreeForAll")
		{
			if (GEngine)
			{
				GEngine->AddOnScreenDebugMessage(
					-1,
					10,
					FColor::Yellow,
					FString::Printf(TEXT("Joining match type %s"), *MatchType)
				);
			}

			// Steam complains about bUseLobbiesIfAvailable and bUsesPresence matching
			// so changing these in the results is unfortunately required. At least
			// it's a pretty simple workaround. Will likely need more testing in a 
			// production environment but this works for now.
			Result.Session.SessionSettings.bUseLobbiesIfAvailable = true;
			Result.Session.SessionSettings.bUsesPresence = true;

			OnlineSessionInterface->AddOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegate);
			const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
			OnlineSessionInterface->JoinSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, Result);
		}
	}
}
```
If this succeeds, we call `JoinSession()`. This results in the `OnJoinSessionComplete_Callback()` firing which allows us to pull the joiner into the session with the host:

```cpp
void AMenuSystem_MPCharacter::OnJoinSessionComplete_Callback(FName SessionName, EOnJoinSessionCompleteResult::Type Result)
{
	if (!OnlineSessionInterface.IsValid()) return;

	FString Address;
	if (OnlineSessionInterface->GetResolvedConnectString(NAME_GameSession, Address))
	{
		if (GEngine)
		{
			GEngine->AddOnScreenDebugMessage(
				-1,
				10,
				FColor::Green,
				FString::Printf(TEXT("Connect String: %s"), *Address)
			);
		}
	}
	
	APlayerController* PlayerController = GetGameInstance()->GetFirstLocalPlayerController();
	if (PlayerController)
	{
		PlayerController->ClientTravel(Address, ETravelType::TRAVEL_Absolute);
	}
}
```