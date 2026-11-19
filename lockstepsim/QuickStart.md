# Quick start: from the starter template to a two-player match

This page takes you from installing the plugin to two windows playing one match of your own copy of the
starter game. Most of the time goes into C++ builds and the engine's first start-ups, and how long those
take depends on the PC far more than on anything you do (see section 4).

You need Unreal Engine 5.8 on Windows, and a C++ project. If your project is Blueprint-only, add C++ to it
first: in the editor, **Tools > New C++ Class**, pick **None**, accept the name, and let the editor build.

## 1. Install the plugin

- **From Fab:** install Lockstep Sim to your engine from the Epic Games Launcher (Fab Library), then in your
  project open **Edit > Plugins**, search "Lockstep Sim", tick it, and restart the editor.
- **By hand:** copy the `LockstepSim` folder into your project's `Plugins` folder (create the folder if
  needed: `<YourProject>/Plugins/LockstepSim/LockstepSim.uplugin`), then open the project. The editor asks
  to build the plugin's modules: answer **Yes**. Or build from a command line first (the first build of a
  project compiles the plugin and your module from scratch, so it is the longest one; how long depends on the PC):

  ```
  "<Engine>\Engine\Build\BatchFiles\Build.bat" <YourProject>Editor Win64 Development -Project="<path>\YourProject.uproject" -WaitMutex
  ```

  `<Engine>` is where Unreal is installed, for example `C:\Program Files\Epic Games\UE_5.8`. The build
  prints many C4996 deprecation warnings from the engine's own headers, "Visual Studio compiler ... is not
  a preferred version", and `[Adaptive Build] Excluded from LockstepSim unity file: ...` (and the same for the
  plugin's other modules: Unreal compiles files it treats as being worked on one by one instead of in bundles).
  All of them are harmless and come with Unreal, not with the plugin.

The plugin has four modules. **LockstepSim** (the framework) and **LockstepUI** (the host / join screen) are what
your game uses. **LockstepStarter** and **LockstepPlayground** are examples: they load in the editor (and in games
started from it), and are never built into a packaged game, so they cost your shipped game nothing.

## 2. Try the starter as it is

1. In the Content Browser, open **Settings** (the gear) and tick **Show Plugin Content**. Open
   `LockstepSim Content > Starter > L_Starter`.
2. Play it in two windows: open the **Play** button's menu (the three dots next to it) and choose
   **Standalone Game**; do it once more for a second window. (Lockstep does its own networking, so Unreal's
   own client / server net modes are not used: each window is a plain standalone game.)
3. In the first window the **Multiplayer** screen is open on the right: press **Host**. In the second window
   press **Join** (the address is already 127.0.0.1, which means "this PC").
4. Both windows now list two players, each with a ping and "in sync". **Left-click** spawns a unit,
   **right-click** sends your units to the cursor. Each player's units have their own colour, and both
   windows show exactly the same thing. **M** hides or shows the Multiplayer screen.

### First, once

`UnrealEditor.exe` (and `UnrealEditor-Cmd.exe`, used for the tests) are in `<Engine>\Engine\Binaries\Win64\`;
run them from there or with that path in front.

Before you start any game from a command line, run the starter's tests once (the command below). The first
starts of a project that the editor has never opened are slow: every `-game` start scans the project's assets
again, because only the editor saves that scan, and on a large project or a slow disk that takes minutes. During the scan the log goes quiet (the
last plugin line is `Network: hosting ...` or `Network: joining ...`); the game has not hung, and the
`Status:` lines start once the scan is done. Opening the project in the editor once, or running any
`UnrealEditor-Cmd.exe` command on it once, saves the scan, and later starts skip it (the first start after a
build still loads the new code, see section 4). The tests' own first run does the scan and loads the freshly
built code, so it is slow too: it can take several minutes, depending on the PC. Let it finish. The console shows only a few build-tool lines, and some
look like failures: `##PlatformValidate: <platform> INVALID ...` and `Unable to find required SDK version ...
for platform <platform>`, for any platform whose SDK you have not installed (Android, IOS, Mac, Linux, TVOS,
VisionOS, ...). That is the engine checking SDKs; it is harmless. The results are in
`Saved/Logs/<YourProject>.log`, which ends with `**** TEST COMPLETE. EXIT CODE: 0 ****` (the three tests'
`Test Completed` lines say `Result={Success}`). The exit code (0) is what counts: the engine ends the run
abruptly, and now and then the last lines of the log are lost (see [section 6](#6-run-the-tests)). The command:

```
UnrealEditor-Cmd.exe "<path>\YourProject.uproject" -ExecCmds="Automation RunTests LockstepStarter.Rules; Quit" -unattended -nullrhi
```

### From a command line

You can start the same two windows without clicking Host and Join:

```
UnrealEditor.exe "<path>\YourProject.uproject" /LockstepSim/Starter/L_Starter -game -windowed -ResX=960 -ResY=600 -LockstepHost
UnrealEditor.exe "<path>\YourProject.uproject" /LockstepSim/Starter/L_Starter -game -windowed -ResX=960 -ResY=600 -LockstepJoin=127.0.0.1
```

Add `-LockstepName=Anna` for a player name, and `-LockstepFakeLag=100 -LockstepFakeJitter=30` to feel a slow
connection on one PC.

`UnrealEditor.exe` (and a packaged game's `.exe`) is a windowed program: started from a command prompt it
returns at once and keeps running on its own, which is what you want for two games side by side. To wait for
one to finish and get its exit code (in a script, or before reading a file it writes), start it with
`start /wait "" "<exe>" ...` in a batch file (`%ERRORLEVEL%` is then the game's exit code), or in PowerShell:

```
$p = Start-Process -FilePath "<Engine>\Engine\Binaries\Win64\UnrealEditor.exe" -ArgumentList '"<path>\YourProject.uproject"', '-game', '-nullrhi' -Wait -PassThru
$p.ExitCode
```

(`UnrealEditor-Cmd.exe`, used for the tests, is a console program and waits by itself.)

### With no screen (a script, a build server)

Add `-nullrhi` (no window; run the starter's tests once first, see [First, once](#first-once)), and any of these:

- `-LockstepLogStatus=5`: the sync status in the log every 5 seconds (`Status: NETWORK: in sync with the
  other machine for 349 checked ticks ... you are player 1 of 2`), the players list with names, pings and
  sync state (`Status: PLAYERS: 1 Anna (host, you, in sync) | 2 Ben (12 ms, in sync)`; right after joining,
  before the first checksum has been compared, these say `joined, not checked yet` and `NETWORK: joined the
  match, no tick compared ... yet`), and the checksum
  after every 100th tick (5 seconds of ticks): `Checksum after tick 399: 0x...`. Tick numbers are the same on
  every machine, so the same line in both games' logs must show the same checksum. (The checksum inside the
  `Status: LOCKSTEP` line is the newest one this machine has, at whatever tick it was written: compare the
  `Checksum after tick` lines, not those.) The checksum is taken after the tick ran, so it includes that tick's
  orders; with `-LockstepLogOrders`, a tick's `Order on tick` lines come before its `Checksum after tick` line.
- `-LockstepLogOrders`: every order handed to the rules, with its tick, name, values (written the way the
  order script writes them, exactly: a Fixed value is a whole number, a short decimal such as `1.5`, or a
  fraction such as `21845/65536` for 1/3, so a logged line pasted into an order script gives the same order),
  order type (the number from your order enum) and player (`Order on tick 103:
  "Move Units" 2000 8000 0 (order type 2) from player 2, handed to the rules.`). Every machine logs the same orders on
  the same ticks; whether an order changed anything is up to your rules (they may refuse its values).
- `-LockstepLogState=<filter>`: with each status write, the state fields whose names contain the filter
  (`State at tick 400: unit 0 / position = ...`): see what an order did. The names are the ones the desync
  report uses. Names have spaces, so put the filter in quotes: `-LockstepLogState="unit 0 /"` (without them
  the command line keeps only the first word). The game says once which filter it uses:
  `-LockstepLogState: every 5 s, the state fields whose names contain "unit 0 /".`, and warns when words
  after the filter look like the rest of one that lost its quotes. A joining game writes state lines once it
  has the host's game (before that it holds only its own start-up world). In a window, the console command `Lockstep.State <filter>` does the same once. Each game
  writes these lines on its own status timer, so two games' state lines are at different ticks: compare
  machines with the `Checksum after tick` lines, and read state lines for what one game holds. Positions
  in state lines are the simulation's own: `chunk (0, 0), local (2621440, 2621440) raw, height 0 raw (about
  40.000, 40.000 m in the chunk)`. The map is cut into 256 m chunks, "local" counts 1/65536 m inside the
  chunk, and the part in brackets is the same in metres. So the position in metres is chunk x 256 + the
  metres in the chunk (here 40 m, 40 m), and an order that said `4000 4000 0` (centimetres) lands there.
- `-LockstepOrderScript=<file>`: orders from a text file, one per line, `<seconds> <order name> <values>`
  (`#` starts a comment). With `-LockstepHost` / `-LockstepJoin` the seconds count from the start of the
  match. The values follow the order's parameters (as registered for Blueprints), in order:

  | Parameter kind | Written as | Example |
  |---|---|---|
  | Position | three numbers, X Y Z in cm | `2000 3000 0` |
  | Int, Int64 | a whole number | `4`, `-12` |
  | Bool | `true` / `false` (or `1` / `0`) | `true` |
  | Fixed | a fraction or a decimal | `3/4`, `2.5` |

  In the starter's top-down view, +X points up the screen and +Y to the right (the camera looks straight down
  with +X at the top).

  ```
  # spawn two units, then send them 50 m along +Y (to the right on the starter's screen)
  1 Spawn Unit 2000 3000 0
  1 Spawn Unit 2200 3000 0
  4 Move Units 2000 8000 0
  ```

  An order whose name has spaces needs no quotes, and names are not case-sensitive. When the game starts,
  the log says `-LockstepOrderScript: 3 order(s) from <file>, timed from the start of the network match.`,
  and `Order script: line 2: issued "Spawn Unit 2000 3000 0" for tick 66.` as each one goes out. If any
  line cannot be read, the game gives none of the file's orders and the log says which line and why.
- `-LockstepQuitAfter=60`: quit after 60 seconds of play (time spent loading does not count).

For example, two games that meet, play the script above in the joining one, and quit:

```
UnrealEditor.exe "<path>\YourProject.uproject" /LockstepSim/Starter/L_Starter -game -nullrhi -LockstepHost -LockstepLogStatus=5 -LockstepLogOrders -LockstepQuitAfter=60 -abslog="<path>\Host.log"
UnrealEditor.exe "<path>\YourProject.uproject" /LockstepSim/Starter/L_Starter -game -nullrhi -LockstepJoin=127.0.0.1 -LockstepOrderScript="<path>\Orders.txt" -LockstepLogStatus=5 -LockstepLogOrders -LockstepQuitAfter=50 -abslog="<path>\Join.log"
```

Without `-abslog`, each game's log is `Saved/Logs/<YourProject>.log` (the second game writes a numbered
copy). The plugin's lines start with `LogLockstepSim:`; search for that. The rule for reading a log: only a
`LogLockstepSim: Error:` line is a problem from the plugin, and it always says what to do. The engine writes
lines of its own with "Error", "Warning" or "Failed" in them at every start, and none of them are problems.
Examples (the list depends on your engine install and PC): `LogTemp: Error test: UE::UnifiedErrorTest::...`
and `LogTemp: FError that has been invalidated / moved from` (Unreal's start-up self-checks), about twenty
`LogEditorDataStorageUI: Warning: Unable to register widget factory ...`, `LogWindows: Failed to load
'aqProf.dll'` (also `VtuneApi.dll`, `WinPixGpuCapturer.dll`) and `PixWinPlugin: PIX capture plugin failed to
initialize!` (profiler tools that are not installed), and, when two games start together, `LogFileManager:
Warning: Error deleting file: ...TurnkeyReport_1.log` and `LogCore: Warning: UTS: Unreal Trace Server process
returned an error` (both games reach for the same file or tool). Whatever the engine prints, the rule above
decides: only `LogLockstepSim: Error:` lines need you.

In a game with a window, the console command `Lockstep.Order Spawn Unit 2000 3000 0` gives an order the
same way as a line of an order script.

## 3. Copy the starter into your own game

The starter lives in the plugin so it stays untouched; your game should be a renamed copy in your own module.
Close the editor (the script refuses while the project is open), then run, from a command prompt or PowerShell:

```
powershell -ExecutionPolicy Bypass -File "<YourProject>\Plugins\LockstepSim\Extras\StarterTemplate\CopyStarterToProject.ps1" -Prefix Tank
```

With the plugin in your project's `Plugins` folder (installed by hand), that is all: the script works out the
project from where it is. If Fab installed the plugin into the engine, the script is in
`<Engine>\Engine\Plugins\Marketplace\<plugin folder>\Extras\StarterTemplate\`, and you also name your project:
add `-Project "<path>\YourProject.uproject"`.

`-Prefix` is the name your classes get in place of "Starter" (here: `FTankGame`, `UTankSimComponent`,
`ATankWorld`, `ATankPlayerController`, `ATankGameMode`). It also names your tests (`Tank.Rules.*`); the
test filter in section 6 is `Tank.Rules`, so they never mix with the engine's own tests.

The script:

- copies seven files into `Source/<YourModule>/`: `TankGame.h/.cpp` (the rules), `TankSimComponent.h/.cpp`
  (the simulation component and its named orders), `TankWorld.h/.cpp` (drawing, camera, mouse, game mode)
  and `TankGameTests.cpp` (determinism tests);
- adds the modules they need (`LockstepSim`, `LockstepUI`, `InputCore`, `Slate`, `SlateCore`) to your
  `<YourModule>.Build.cs`, and removes the New Project wizard's commented-out hint for Slate there
  (`// Uncomment if you are using Slate UI` and the commented `PrivateDependencyModuleNames` line under it),
  since Slate is now really added;
- enables the plugin in your `.uproject`;
- makes `TankGameMode` your project's default game mode (in `Config/DefaultEngine.ini`), if the project has
  none yet, as in a new project. If yours already has one, the script leaves it and says so; then set the
  level's game mode yourself (section 4). `-NoDefaultGameMode` skips this.

It never overwrites an existing file, and its output lists every change it made. If you prefer to do it by
hand, those four bullets are everything it does: copy the seven files `StarterGame.h`, `StarterSimComponent.h`
and `StarterWorld.h` (from `Source/LockstepStarter/Public`) and `StarterGame.cpp`, `StarterSimComponent.cpp`,
`StarterWorld.cpp` and `StarterGameTests.cpp` (from `Private`), leaving out `LockstepStarterModule.cpp` (the
starter module's own start-up file: your module already has one), replace `Starter` with your prefix
and `LOCKSTEPSTARTER_API` with `YOURMODULE_API` (your module's name in capitals), add the modules, and set
**Project Settings > Maps & Modes > Default GameMode** to `TankGameMode`.

## 4. Build and play your copy

1. Open the project: the editor offers to build your module. Answer **Yes** (or build from your IDE, or
   from a command line: `"<Engine>\Engine\Build\BatchFiles\Build.bat" <YourProject>Editor Win64 Development -Project="<path>\YourProject.uproject" -WaitMutex`, as in section 1).
   This build can take as long as the first one, or longer: the new files and modules make
   Unreal rebuild the module's makefile and precompiled header ("Creating makefile ... (.uproject file is
   newer)"). After that, how long a build takes depends far more on the PC than on your change: on one PC a
   one-line `.cpp` change built in 9 to 15 s, on another (project and engine on hard drives) the same kind of
   change took 6 to 10 minutes every time. A change that touches a header or several files, such as
   [adding an order](OrdersAndRules.md), recompiles more. A long build is the build tools, not your change, and
   not a sign of a problem: let it finish. Most builds print
   `Requested 1.5 GB memory per action, N GB available: limiting max parallel actions to M`; that is normal.
   Only a small M (1 or 2) means the PC is short of free memory and compiles few files at once: close other
   programs (a running editor or game) to speed it up.
2. Play it in two windows exactly as in section 2, in whatever level is open (the game mode adds a `TankWorld`
   to any level by itself). Host in one, join in the other. A new project's level is the engine's
   `OpenWorld` template map, which is fine; to start in another one, set **Project Settings > Maps & Modes >
   Editor Startup Map** and **Game Default Map**.

The first game start (or test run) after a build is slow: the engine loads the new code and checks its files
first, which can take minutes on some PCs, even with `-LockstepQuitAfter` (it counts seconds of play, not
loading). Later starts skip that work.

If your project already had a default game mode, open a level first and set **World Settings > GameMode
Override** to `TankGameMode`.

From a command line, the project's default map works as it is (and every switch from
[With no screen](#with-no-screen-a-script-a-build-server) works with your copy too):

```
UnrealEditor.exe "<path>\YourProject.uproject" -game -windowed -ResX=960 -ResY=600 -LockstepHost
UnrealEditor.exe "<path>\YourProject.uproject" -game -windowed -ResX=960 -ResY=600 -LockstepJoin=127.0.0.1
```

(Any other level or game mode can be named in the URL: `"/Engine/Maps/Entry?game=/Script/<YourModule>.TankGameMode"`.)

Your copy plays exactly like the starter (same rules): the same orders on the same ticks give the same
checksums. Two separate matches are not that, even with one order script, because the script counts seconds
from the start of the match and matches start on different ticks; the proof's values are the real
comparison (in [ProofTool](ProofTool.md), a fresh copy's values equal the starter's). To see that your own module is
the one running, look in the log for `LogLoad: Game class is 'TankGameMode'` (your prefix; the plugin's own
starter says `StarterGameMode`).

You now have a multiplayer lockstep game of your own. Everything else is growing it.

## 5. Add your first order

[Adding your own orders and rules](OrdersAndRules.md) walks through adding a "Stop Units" order end to end:
the order number, its builder, applying it in the rules, naming it for Blueprints, issuing it from a key,
and testing it.

## 6. Run the tests

Your copy includes three automation tests (`TankGameTests.cpp`), named after your prefix (`Tank.Rules.*`),
and a workload for the cross-machine [proof tool](ProofTool.md) in the same file.
In the editor: **Tools > Session Frontend**, the **Automation** tab, filter `Tank.Rules` (your prefix, then
`.Rules`: the filter matches any test name that contains it, and only your tests' names contain that), tick the
tests, **Start Tests**. From a command line:

```
UnrealEditor-Cmd.exe "<path>\YourProject.uproject" -ExecCmds="Automation RunTests Tank.Rules; Quit" -unattended -nullrhi
```

The results are in `Saved/Logs/<YourProject>.log` (search for `Test Completed`: one line per test, with
`Result={Success}` or `Result={Fail}`). The command's exit code is the result: 0 when every test passed, not 0
(for example 255) when one failed. When every test passes, the log ends with `**** TEST COMPLETE. EXIT CODE: 0 ****`
(`EXIT CODE: -1` when one fails). The engine ends the test run abruptly, though, and now and then the last lines
of the log never reach the file: the TEST COMPLETE line, and sometimes the last test's `Test Completed` line
too. A log that stops early is not a failure; trust the exit code, and run the tests again if you want to read
those lines. A failure always shows in the log as `LogAutomationController: Error: Test Completed. Result={Fail} ...`
with the failed check's text next to it (for example `Expected 'every unit reached its spot and stopped' to be true`).
The file and line number next to it are only approximate: the line can be the one just after the check, and two
checks next to each other can show the same line. Find the failure by the check's text, so write texts that say
what was expected. Add a test for every rule you add (same seed and same orders must give the
same checksum): copy the `MakeScript` / `RunScript` pattern in `TankGameTests.cpp`. The command line also
writes the asset scan the editor would (see [First, once](#first-once)), so
later game starts skip the scan. The first test run (or game start) after a rebuild is still slow while the
engine loads the new code and checks its files (minutes on some PCs); the runs after it are much shorter.

## What next

- [Core concepts](Concepts.md): what is going on under the hood.
- [Troubleshooting](Troubleshooting.md): when the two windows stop agreeing.
- The Playground (`LockstepSim Content > Playground > L_Playground`): thousands of units and a guided
  eight-lesson tutorial of every feature.
