# The proof tool: is your game deterministic across computers?

Lockstep only works if every player's computer computes the simulation bit for bit identically. The
plugin is built for that (fixed-point math, its own random generator, no engine state in the rules), and
the proof tool lets you check it for *your* game on *your* players' kind of hardware: run the same build
on two computers and compare the results.

It needs no second copy of the editor and no network: each computer runs the proof on its own and writes
a text file; you compare the two files.

## What it runs

- The plugin's own self-tests (math, positions, saves, replays, networking over an in-memory connection,
  the desync finder...) and their recorded checksums.
- A math workload: 200,000 rounds of fixed-point arithmetic, square roots, random numbers and positions
  up to 2,000 km apart.
- Every workload your game adds (see below): your rules, run with scripted orders, checksummed.

Every result is written as `key = value` lines. `machine.*` lines say where the file came from;
everything else must be identical on every computer that runs the same build.

## 1. Add your rules to the proof

A copy of the starter template already has a workload: the end of your `<Prefix>GameTests.cpp` plays an
eight-player match for 3,000 ticks and records its checksum every 1,000 ticks
(`checksum.<Prefix>RulesTick1000` ... in the results) and one hash of every tick's checksum
(`checksum.<Prefix>RulesAllTicks`). Grow it as your rules grow: add each new order to its script, run the
proof before and after, and check that `<Prefix>RulesAllTicks` changed. Take the "before" run after your
rule and state changes but before you add the order to the workload: a new state field that the checksum
covers changes every `<Prefix>Rules*` value by itself, so a "before" from earlier would show a change even if
the workload never gave the order. Checksums at a few ticks are not
enough on their own: an order whose effect a later order overwrites (units sent somewhere, then moved on
before the next checkpoint) leaves them unchanged, so the proof would not cover the new rule. The every-tick
hash changes whenever the order changes anything at any tick. In a workload of your own, do the same (an
`FSimChecksum` fed every tick's `ComputeChecksum()`).

A workload can live in any `.cpp` file of your game; no module class is needed:

```cpp
#include "LockstepProof.h"
#include "LockstepTestKit.h"
#include "LockstepWorld.h"

LOCKSTEP_PROOF_WORKLOAD(MyRules, "My rules")
{
	FLockstepWorld World(MakeUnique<FMyGame>(FMyRules()), 12345);
	// ... step 5,000 ticks with scripted orders ...
	FLockstepTestRecord::AddValue(TEXT("MyRulesTick5000"), World.ComputeChecksum());
}
```

The first argument names the workload's function (unique in your game), the second is the name the proof
prints. The workload is stored when your module loads and runs only when the game is started with
`-LockstepProof`. (From a module class of your own you can also call `FLockstepProof::AddWorkload(Name,
Function)` in `StartupModule`.)

The scripted orders are the same kind your automation tests use: the starter's workload shows how, with the
`WithPlayer` / `RunScript` helpers at the top of `<Prefix>GameTests.cpp`. Make the workload big enough to exercise your rules (many units,
long enough to use every system) and make its orders depend only on the tick number, never on time. The proof
needs no level or game mode (it runs as soon as the engine has started), and your automation tests are not
part of it: put what you want compared between computers in workloads. If your game runs its own
`FLockstepTestGroup` groups at startup, call `FLockstepTestRecord::ExpectGroups(Count)` before running them,
so a group that never reports makes the proof fail.

## 2. A quick look (one computer, no packaging)

To see the results file on this computer while you work, run the proof from the editor's executable:

```
start /wait "" "<Engine>\Engine\Binaries\Win64\UnrealEditor.exe" "<path>\YourProject.uproject" -game -nullrhi -LockstepProof -LockstepProofOut="<path>\Results.txt"
```

(`start /wait` makes the command wait for the game; in a batch file `%ERRORLEVEL%` is then its exit code, 0
when everything passed. The editor is a windowed program and otherwise returns at once. In PowerShell use `Start-Process ... -Wait
-PassThru` and read `.ExitCode`.) How long it takes depends on your workloads and the PC. The first run after
a rebuild is slower (the engine loads the new code first), and so is a run on a project that has never been
opened (the first-start asset scan, see the Quick start). Two things differ from a packaged run, so never compare an editor run with a packaged
one (the compare tool says so): `build.Target` is `Editor`, and the plugin's example modules are loaded,
which adds the starter's own `checksum.StarterRules*` values. To compare computers, use step 3.

Two more things to know about a quick look:

- **`build.ExeMD5` is the editor's**, the same for every build of your game. To tell two builds of your game
  apart, look at `build.CodeMD5`: a fingerprint of your game's own code (every module file under your
  project folder: your game module and the plugins in your project's `Plugins` folder). Every rebuild gives
  a new value, even of unchanged code (the compiler does not write byte-identical files twice), so two runs
  of one build share it, and any rebuild in between changes it. The compare tool notes it when the two files
  differ there. If only `build.*` lines differ and every `group.*` and `checksum.*` line matches, your rules
  agree: the builds differ, the results do not (the tool says so too). In a packaged game the
  code is inside the executable, so the line says `in the executable` and `build.ExeMD5` covers it. Even so,
  compare quick looks only with quick looks of the same build: a rule change shows as a checksum difference,
  not as a determinism problem.
- **A fresh copy of the starter gives the same values as the starter.** Right after
  `CopyStarterToProject.ps1`, your `checksum.<Prefix>Rules*` values equal the `checksum.StarterRules*` values
  in the same file: the copy runs the same rules on the same script. That is expected, and it shows your copy's
  workload is running. They part as soon as you change a rule or add an order to the workload.

## 3. Package and run

1. Package your game (Development is fine): in the editor **Platforms > Windows > Package Project**, or from
   a command line:

   ```
   "<Engine>\Engine\Build\BatchFiles\RunUAT.bat" BuildCookRun -project="<path>\YourProject.uproject" -platform=Win64 -clientconfig=Development -build -cook -stage -pak -archive -archivedirectory="<path>\Packaged"
   ```

   The game is then in `<path>\Packaged\Windows\`.
2. Copy `RunProof.bat` from the plugin's `Extras/ProofTool/` folder into the packaged game's top folder
   (next to `<YourGame>.exe`; a `Game\` folder next to the .bat works too, or drag the .exe onto it).
3. Double-click `RunProof.bat`. It runs the game with no window (`-LockstepProof -nullrhi`), waits
   (how long depends on your workloads and the PC), and writes `LockstepProof_<computer name>.txt` next to itself. The last line says
   `overall = PASS` if every test passed on this computer.
4. Copy the same packaged folder to the other computer and do the same there. Nothing needs installing
   (if Windows asks for the Visual C++ runtime, install Microsoft's "Visual C++ Redistributable", or
   ship it with your game as Unreal's packaging settings allow).

You can also run it by hand: `start /wait "" <YourGame>.exe -LockstepProof -nullrhi -LockstepProofOut=C:\results.txt`
(without `start /wait`, or `Start-Process -Wait` in PowerShell, the command returns before the game has
finished). The exit code is 0 when everything passed. In a script, `RunProof.bat /nopause` does not wait for a key
at the end (exit code 0 = passed, 1 = failed, 2 = no game found).

## 4. Compare

Put both results files in one folder with `CompareResults.bat` and `CompareResults.ps1` (from
`Extras/ProofTool/`) and drag the two files onto `CompareResults.bat`. Double-clicking it instead finds the
results files in its folder by their contents, whatever they are called (`Results.txt`,
`LockstepProof_<computer>.txt`, ...): it compares a `Reference*.txt` results file if there is one, else the
newest, with the newest results file from a different computer. Two runs on the same computer are not picked
by a double-click; drag both onto it (or pass them) to compare those.

The tool compares every line except `machine.*`: the `group.*` lines, the `checksum.*` lines, the `build.*`
lines and `overall`. So its "N values compared" is larger than the "groups, checksums" counts in the
`overall` line, which count only those two kinds.

The last line says **MATCH** (every value identical: the two computers compute your game identically),
**MISMATCH at: ...** with the values that differ, or **NOTHING TO COMPARE** when the second file is not
there yet. In a script: `CompareResults.bat /nopause <file1> <file2>`, exit code 0 = match, 1 = mismatch,
2 = nothing to compare.

## Reading a mismatch

1. **`build.ExeMD5` or `build.CodeMD5` differs**: the two runs did not use the same build. Lockstep needs
   the same build everywhere; compare runs of one build. (If `build.Target` differs too, one file is a quick
   look from the editor: compare two runs of the packaged game. Two quick looks always share `build.ExeMD5`,
   the editor's; `build.CodeMD5` is the one that tells their builds apart.)
2. **Only `ProofMathSweep` or the plugin's math checksums differ**: the computers do not agree on basic
   integer math. This should never happen on supported platforms; report it with both files.
3. **Only your workload's checksums differ**: your rules read something that differs between computers
   (a float, the clock, an engine value, uninitialised memory). Run your workload twice on one computer
   with Run Second Copy on, or record a replay, to find the field (see [Troubleshooting](Troubleshooting.md)).
4. **Only `group.TcpTransport` fails**: that computer's firewall or antivirus blocked a local TCP
   connection; it says nothing about determinism.
