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

## Accessing the subsystem
We use sessions to handle our multiplayer connections, and the `OnlineSubsystem` comes with an interface, `IOnlineSessionInterface`. We can store that interface like so:

```cpp
// Pointer to the online session interface
TSharedPtr<class IOnlineSession, ESPMode::ThreadSafe> OnlineSessionInterface;
```

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