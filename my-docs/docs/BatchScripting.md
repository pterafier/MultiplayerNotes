# Test Multiplayer Locally

Using a batch script, we can test multiplayer code without needing to package, zip, and download the project to another machine. Instead, we can simply run two standalone instances of the game using the following batch script:

```bat
"F:\UE_5.5\Engine\Binaries\Win64\UnrealEditor.exe" "G:\UnrealProjects\MenuSystem_MP\MenuSystem_MP.uproject" -game -ResX=960 -ResY=540 -log -WINDOWED -nosteam
```
>The first path leads to your current Unreal version's .exe of the editor which should be fairly close to what's shown above, and the second path is the path to your current project. `-nosteam` sets the OnlineSubsystem to NULL which allows you test locally

## Creating a batch script

To create a batch script, right-click in the file explorer and select "New text file", then change the `.txt` to `.bat`. Edit the file in notepad or a text editor with the above script and open the batch script twice. This will generate 2 game instances that can connect to each other.