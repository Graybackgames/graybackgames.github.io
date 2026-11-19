# Troubleshooting

Most problems with a lockstep game are desyncs, so most of this page is about finding and fixing them.
The rest covers connecting, stutter, saves and replays.

## Finding a desync

### 1. Catch it early, on one machine

Keep **Run Second Copy** on while you develop (Project Settings > Plugins > Lockstep Sim; it is on by
default). The plugin then runs two copies of your world side by side and compares them after every
checked tick. Most determinism bugs (a float in the rules, a `TMap` loop, a random number from the wrong
place) make the two copies drift apart within seconds, on your own PC, without a second machine.

Turn on the overlay (Project Settings > Plugins > Lockstep Sim > Show Overlay, or the component's
Show Overlay, or the console command `LockstepSim.Overlay 1`) to see the sync state at all times.

### 2. Read the report

When a desync is caught the plugin writes a report:

- **Local copies** (Run Second Copy): to the log, as an error starting with `DESYNC FOUND`.
- **Between machines**: to the log on both machines, and to `Saved/Desync/Desync-<time>-tick<N>-player<P>.txt`
  on both machines. In Blueprints, the Lockstep Sim component's **On Desync** event fires with a one-line
  summary.

A report looks like this:

```
DESYNC FOUND: the game's two copies (copy A and copy B) stopped agreeing on tick 412.
They ran the same orders and agreed on every tick before it, so this is the first tick where they differ.
  copy A: checksum 0x3a9c0d51e2f7b604
  copy B: checksum 0x91fe0c2d74ab1e38
First difference: unit 17 / position: copy A = chunk (0, 0) local (3276800, 655360) z 0, copy B = ...
3 fields differ in total, in the order the game lists its state; the first is the likeliest cause, the rest are usually knock-on effects:
  - unit 17 / position: ...
  - unit 17 / target: ...
  - unit 17 / moving: ...
What this means: ...
What to do: ...
```

Read it like this:

1. **The tick.** Everything agreed before it. The bug ran on that tick (in `Tick`, or in `ApplyOrder` for an
   order that ran on that tick).
2. **The first difference.** Fields are listed in the order your `DescribeState` lists them, which is the
   order your rules update them in, so the first one is usually the cause and the rest follow from it.
3. **The values.** A tiny difference (one raw unit) usually means rounding: a float somewhere. A large or
   unrelated difference usually means different logic ran: an iteration order, an uninitialised value, or
   different builds.

If the report says **"different rules"** (a field under `rules / ` differs), the two machines are not
running the same game: different versions, mods or settings. Every machine needs the same build and the
same project settings (especially Ticks Per Second).

If the report says **"every field the game describes is equal, yet the checksums differ"**, your
`DescribeState` leaves out something that `AddStateToChecksum` hashes. List every hashed field in
`DescribeState`, in the same order, and the next report will name it.

If the report says **"which field differs could not be found"**, the machine kept no history for that tick
(see Net Desync History Ticks on `FLockstepSession`); the tick and checksums are still right.

### 3. Reproduce it

Record a replay (`StartRecording` / `StopRecording` on the component, or `FLockstepSession::BeginRecording`).
Stopping a recording plays it back on a separate world and says whether it played back identically. A
replay that does not play back identically reproduces the bug on demand, as often as you like, on one
machine.

## The usual causes (and their fixes)

| Symptom in the report | Likely cause | Fix |
|---|---|---|
| A position or speed off by a tiny amount | `float` / `double` math in the rules, or `FMath` functions that use them (`FMath::Sqrt`, `Sin`, `Atan2`) | Use `FSimFixed`, `FSimPos`, `FSimMath`, `FSimTrig`, `FSimVec2`. Floats are for drawing only. |
| Different objects chosen ("nearest enemy" differs) | Looping over a `TMap` or `TSet`, sorting with an unstable or incomplete comparison, pointers used as keys | Keep objects in `TArray`s by index; break every tie with the id. |
| Values that differ after a load or a join | A field not written by `SaveState` (or not read back by `LoadState`) | Save and load every field you hash, in the same order. |
| Random behaviour differs | `FMath::Rand`, `FRandomStream`, `std::rand` or time-based seeds | Use `Context.Rng` inside `Tick` / `ApplyOrder` only. |
| A difference right after an order from Unreal | The order was built from a float on each machine, or read engine state while running | Convert floats to sim values once when *issuing* the order (`FLockstepQuantize`, the Blueprint Add nodes); never read actors, physics, the clock or frame time inside the rules. |
| A difference only on some PCs | Different builds (or a compiler setting such as fast floating point) | Everyone runs the same build. Run the proof tool on both PCs (see [The proof tool](ProofTool.md)). |
| Garbage values in a new field | Uninitialised memory | Give every field a default value. |

## Fixing it during a match: Repair

A desynced player can be repaired without ending the match: press **Repair** on the host / join screen
(the host repairs every out-of-sync player; an out-of-sync player repairs itself), or call `RepairMatch` /
`FLockstepNetLink::RepairOutOfSync` / `RequestRepair`. The player receives a fresh copy of the host's world
and the match carries on. This hides the symptom; the bug is still there, so keep the report.

## Connecting

| Problem | What to do |
|---|---|
| "No host answered" | Is the host running and hosting? Same port on both? On one PC use 127.0.0.1. |
| Other computers cannot join | The host must tick **Allow other computers** (or set Bind Address to 0.0.0.0 in the project settings), allow the game through the Windows firewall, and the joining player must use the host's LAN address (`ipconfig` shows it). Over the internet the host also needs a port forward for the TCP port. |
| "Different game version" / "different rules or mods" / "different tick rate" | Everyone must run the same build with the same settings. |
| "The match is full" | The host's Max Players (2 to 8) is reached. |
| "A player of this match lost their connection" (asked to wait) | A missing player may still come back; you get a seat once they are back or dropped (Pause Timeout). |

## Stutter and waiting

A tick runs only once every player's orders for it have arrived. If the game stutters in a match:

- Keep **Adaptive Input Delay** on: the host raises the delay when a connection is slow.
- Or raise **Input Delay Ticks**: it should cover the slowest player's one-way lag (through the host for
  player-to-player orders).
- The overlay's network line shows how often this machine waits for the others, the measured round trip
  and the current input delay.

If the game stutters alone, look at the cost of a tick: a tick must finish well inside 1 / Ticks Per
Second (50 ms at 20). Turn off Run Second Copy and raise Checksum Interval Ticks in a shipped game.

## Saves and replays refused

A save or replay is refused (with the reason and what to do) when it is damaged, cut short, from a
different version of the file format, or made with different rules or a different tick rate. A rules
change (any tuning number) makes older saves incompatible by design: the fingerprint protects matches
from mixed rules.
