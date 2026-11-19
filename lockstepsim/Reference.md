# Reference

The public API of Lockstep Sim: the C++ classes by job, then every Blueprint node and event. The header of
each class has the full comments; this page says what each piece is for and how the pieces fit.

Contents: [Numbers and positions](#numbers-and-positions) · [Your game](#your-game) ·
[Running a simulation](#running-a-simulation) · [Networking](#networking) · [Saves and replays](#saves-and-replays) ·
[Desync finder](#desync-finder) · [Unreal integration](#unreal-integration) · [Host / join screen](#host--join-screen) ·
[Starter template](#starter-template) · [Testing and the proof](#testing-and-the-proof) ·
[Blueprint nodes](#blueprint-nodes) · [Blueprint events](#blueprint-events) · [Command line](#command-line) ·
[Console commands](#console-commands) · [Project settings](#project-settings)

Player numbers in the API start at 0 (0 = the host, or the only player when alone). Screens count from 1.

## Numbers and positions

All simulation math uses these types. They are plain integers underneath: the same result on every CPU.

| Type | Header | What it is |
|---|---|---|
| `FSimFixed` | SimFixed.h | A number with 1/65536 precision (Q16.16), range about -32,768 to +32,767. `FromInt`, `FromRatio(Num, Den)`, `FromRaw` (raw = 1/65536), `FSimFixed::Zero()` and `FSimFixed::One()`; the public `Raw` member is the value in raw steps, `FSimFixed::One64` (65536, an `int64`) is the raw value of 1.0; `FloorDiv(A, B)` (`int64` division rounding down); `+ - * /`, comparisons; `FSimFixed::Abs(X)` and `FSimFixed::Sqrt(X)` are static (called on the type, not on a value: `FSimFixed::Abs(Delta)`, never `Delta.Abs()`). Rounds down and saturates (never wraps). There is deliberately no conversion *from* float; `ToFloat` / `ToDouble` exist for drawing only. |
| `FSimSaturationAlarm` | SimFixed.h | Counts every saturation and logs a warning (at most once a second) in development builds, so a value hitting its limit is never silent. A test that saturates on purpose wraps it in `FSimSaturationAlarm::FScopedQuiet`. |
| `FSimPos` | SimPos.h | A position on maps of any size: a 256 m chunk (int32 X, Y) plus an `FSimFixed` offset inside it, and a height. `FromWorldMeters`, `FromWorldRaw(X, Y)`, `Make`; `Offset`, `StepToward(Target, Step)`; `WorldRawX()` / `WorldRawY()` and `DeltaRawX(From, To)` / `DeltaRawY` (exact `int64` in raw units, 1/65536 m); `DistSq256`, `Distance` (exact 64-bit math). For "which is nearer" at any range: `FSimPos::CompareDistance(Origin, A, B)` (static) returns -1 when A is nearer to Origin, +1 when B is nearer, 0 when equal at 1/256 m (break ties yourself, for example by the lower id); `FSimPos::IsWithin(A, B, Range)` (static) is true when B is within `Range` (an `FSimFixed` in metres, horizontal) of A. |
| `FSimRng` | SimRng.h | The simulation's random generator (PCG32). In rules, use the one in the tick context: `Context.Rng.RangeInt(Lo, Hi)`, `NextUnit()`, `RangeFixed(Lo, Hi)`. |
| `FSimChecksum` | SimChecksum.h | FNV-1a 64-bit hash fed field by field (`AddInt32`, `AddInt64`, `AddUInt8`, `AddBool`, `AddFixed`, `AddPos` via `FSimPos::AddToChecksum`). |
| `FSimRate` | SimRate.h | A rate of whole things: `PerSecond(2, TicksPerSecond)` = 2 per second, exact, with an `int32` progress counter you keep in your state: `Rate.Advance(Progress)` returns how many are due this tick. |
| `FSimPerSecond` | SimRate.h | Converts per-second values to per-tick values once: `PerTick(5, 1, TicksPerSecond)` = 5 m/s as metres per tick; `TicksFor(Num, Den, TicksPerSecond)` = a duration in ticks. |
| `FSimAngle` | SimMath.h | An angle as a 32-bit turn (wraps naturally). `FromDegrees`, `FromTurns`, `ToDegrees`, `SignedDelta`, `MoveToward`. Angle 0 points along +X, and a positive angle turns counter-clockwise, from +X towards +Y (`FSimVec2::Rotated` and `FromAngle` use the same direction). In the starter's top-down view (+X up the screen, +Y to the right) that looks clockwise on screen. |
| `FSimMath` | SimMath.h | `ISqrt` (exact for any 64-bit number), `Sqrt`, `Hypot`. |
| `FSimTrig` | SimMath.h | `Sin`, `Cos`, `Tan`, `Atan2` from lookup tables (within 1/65536). |
| `FSimVec2` | SimMath.h | 2D fixed-point vector: `+ - *`, `Dot`, `Cross`, `Length`, `Normalized`, `ScaledToLength`, `ClampLength`, `Rotated`, `Perp`, `FromAngle`, `ToAngle`, `Lerp`. From two positions: `FSimVec2(FSimFixed::FromRaw(FSimFixed::Saturate(FSimPos::DeltaRawX(A, B))), ...)`, for distances under 32 km (see OrdersAndRules, "Your own position math"). |
| `FLockstepQuantize` | LockstepQuantize.h | The one doorway from Unreal floats into the simulation: `CmToSimPos(FVector)`, `CmToFixed(double)`; and back for drawing: `SimPosToCm`. Use it only when building an order. |

## Your game

| Type | Header | What it is |
|---|---|---|
| `ILockstepGame` | LockstepGame.h | The interface your rules implement: `Clone`, `ApplyOrder`, `Tick`, `AddStateToChecksum`, `AddRulesToChecksum`, `SaveState`, `LoadState`, and (for the desync finder) `DescribeState`, `DescribeRules`. See [Adding your own orders and rules](OrdersAndRules.md). |
| `FLockstepTickContext` | LockstepGame.h | Passed to `ApplyOrder` and `Tick`: `Tick` (the tick number), `Rng`, `TicksPerSecond`, and `Emit(Type)` to report an event. |
| `FLockstepOrder` | LockstepOrder.h | An order: `Tick`, `Sequence`, `Player` (stamped by the session), `Type` (yours), `Payload` (bytes). |
| `FLockstepOrderWriter` / `FLockstepOrderReader` | LockstepOrder.h | Write an order's values (`AddInt32`, `AddInt64`, `AddUInt8`, `AddBool`, `AddFixed`, `AddPos`) and read them back in the same order (`ReadInt32`, `ReadInt64`, `ReadUInt8`, `ReadBool`, `ReadFixed`, `ReadPos`; then `IsComplete()`). |
| `FLockstepEvent`, `FLockstepEventWriter` / `FLockstepEventReader` | LockstepOrder.h | Events the rules report during a tick (output only, never state). |
| `FLockstepByteWriter` / `FLockstepByteReader` | LockstepBytes.h | The byte streams used for saves (`SaveState` / `LoadState`); the reader's `ReadCount(MinBytesEach)` guards array sizes. |

## Running a simulation

| Type | Header | What it is |
|---|---|---|
| `FLockstepWorld` | LockstepWorld.h | One copy of a simulation: your game + tick counter + seeded RNG + rules fingerprint. `FLockstepWorld(MakeUnique<FMyGame>(...), Seed, TicksPerSecond)`, `Step(Orders)`, `ComputeChecksum`, `GetTick`, `GetGame<T>()`, `SaveState` / `LoadState`, `Describe`. |
| `FLockstepSession` | LockstepSession.h | Runs a world the way a match does: `QueueOrder(Order)` (returns the tick it runs on), `ConsumeRealTime(Seconds)` (how many ticks are due), `StepOnce()`, `GetInterpolationAlpha()`, `GetWorld()`, `GetLastStepOrders()`, `GetLastStepEvents()`. Settings: `InputDelayTicks`, `bRunSecondCopy`, `ChecksumIntervalTicks`, `NetDesyncHistoryTicks`. Checks: `IsDesynced()`, `GetDesyncReport()`, `GetLastChecked(Tick, Checksum)`. Saves: `SaveGame`, `LoadGame`. Replays: `BeginRecording`, `EndRecording`. Delay: `GetEffectiveInputDelay`, `GetDelayTransition`. Network-match functions are driven by `FLockstepNetLink`. |
| `FLockstepPositionPrediction` | LockstepPrediction.h | Bookkeeping for a unit the player steers directly: which position reports to send, and how far the simulation ended up from what was predicted (`ShouldSend`, `RecordSent`, `Check`, `StartCooldown`). Your drawing code glides out the error. |
| `FLockstepOverlay` | LockstepOverlay.h | The on-screen sync and network status lines (`Draw` every frame, or `BuildLines` for your own widget). Never computes a checksum. Console: `LockstepSim.Overlay 0/1`. |

## Networking

| Type | Header | What it is |
|---|---|---|
| `FLockstepNetLink` | LockstepNetLink.h | Plays a session as a network match of up to 8 players (one host, the others join). Call `Update(Now)` before stepping and `SendClosedBundles(Now)` after. `FSettings`: `Role`, `Address`, `BindAddress`, `Port`, `PlayerName`, `MaxPlayers`, `PauseTimeoutSeconds`, `HandshakeTimeoutSeconds`, `SilenceTimeoutSeconds`, `bAdaptiveInputDelay`, `AdaptiveMaxDelayTicks`, `bSharePlayerList`, `PlayerToken`, `FakeLagMs`, `FakeJitterMs`. State: `GetState`, `IsInMatch`, `IsPaused`, `GetStatusText`, `GetPlayers()` (`FPlayerInfo`: name, host / you, ping, waiting, out of sync, repairing), `GetRoundTripMs`, `GetOutOfSyncMask`, `GetOutOfSyncText`, `GetDesyncReport`. Actions: `RepairPlayer`, `RepairOutOfSync`, `RequestRepair`, `ChangeInputDelay`, `Rejoin`. Callbacks: `OnMatchStarted`, `OnMatchEnded`, `OnMatchPaused`, `OnMatchResumed`, `OnPlayerJoined`, `OnPlayerLeft`, `OnDesyncAlert`, `OnRepairStarted`, `OnRepaired`, `ValidateJoin`. `MakeInstallToken()` gives a stable per-install identity. |
| `ILockstepTransport` | LockstepTransport.h | Whole messages, reliable and in order. Implement it to run the protocol over Steam, EOS or anything else; attach with `FLockstepNetLink::AttachTransport`. |
| `FLockstepTcpTransport`, `FLockstepTcpListener` | LockstepTcp.h | The built-in TCP transport and listener. |
| `FLockstepLoopbackTransport`, `FLockstepFakeLagTransport` | LockstepTransport.h | An in-memory pair (tests) and a wrapper that delays everything by a fixed lag plus random jitter (testing on one PC). |

The host leaving ends the match for everyone; each other machine keeps the world as it was and plays on alone.

## Saves and replays

| Type | Header | What it is |
|---|---|---|
| `FLockstepSession::SaveGame` / `LoadGame` | LockstepSession.h | A whole match, including orders already given but not yet run. Refused (with the reason and what to do) if damaged or made with different rules or tick rate. |
| `FLockstepReplay` | LockstepReplay.h | Start state + every order that ran + the checksum of every (checked) tick. `SaveToFile`, `LoadFromFile`. |
| `FLockstepReplayPlayer` | LockstepReplay.h | Plays a replay on its own world and checks every tick: `Begin`, `StepOnce`, `PlayToEnd`, `IsFaithful`, `GetMismatchTick`, `GetWorld` (for drawing a replay). |

## Desync finder

| Type | Header | What it is |
|---|---|---|
| `FLockstepStateDescriber` | LockstepDesync.h | What `DescribeState` fills: `AddInt`, `AddUInt`, `AddHex`, `AddBool`, `AddFixed`, `AddPos`, with readable paths ("unit 3 / position"). |
| `FLockstepDesyncReport` | LockstepDesync.h | The finder's answer: `FirstDifferingTick`, both checksums, `Diffs` (path and both values, in your listing order), `ToText()` (the plain-language report), `GetOneLine()`, `DiffersInRules()`. |
| `FLockstepDesyncFinder` | LockstepDesync.h | `Capture(World)`, `Compare(WorldA, WorldB)` and `Compare(DumpA, DumpB, ...)`: compare any two worlds yourself, e.g. in a test. |
| `FLockstepStateDump` | LockstepDesync.h | One world's named values at one moment (what machines exchange). |

## Unreal integration

| Type | Header | What it is |
|---|---|---|
| `ULockstepSimComponent` ("Lockstep Sim") | LockstepSimComponent.h | Runs a simulation inside an actor: give it your rules by subclassing and overriding `CreateGame()` (and `RegisterOrderNames()` for named orders). Pumps the network, steps the ticks, fires the events. C++: `GetSession()`, `GetLink()`, `StartWithGame`, `StartNetwork(Settings)`, `RegisterOrder(Name, Type, Params, Description)`, `MakeOrderFromText(Text, Order, Error)` (a named order from text such as `Spawn Unit 2500 4000 0`, as the order script and the console command read it), `LogState(Filter)`, `FindWordsAfterLogStateFilter(CommandLine)` (static: the words left after a `-LockstepLogState=` filter that lost its quotes), `FixedToOrderText(Value)` (static: a Fixed value as an order script writes it, exactly: `1.5`, `21845/65536`). All Blueprint nodes below. Properties: Start On Begin Play, Seed, Show Overlay, Player Name, Accept Command Line, Order Definitions. |
| `ULockstepOrderDefinition` | LockstepBlueprintTypes.h | A Data Asset naming an order for Blueprints: Order Type (number), Order Name, Description, Parameters (`FLockstepOrderParam`: name + kind: Int, Int64, Bool, Position, Fixed). |
| `FLockstepBPOrder`, `FLockstepBPEvent`, `FLockstepBPReader`, `FLockstepBPPlayerInfo` | LockstepBlueprintTypes.h | Orders, events, readers and player rows as Blueprints see them. |
| `ULockstepBlueprintLibrary` | LockstepBlueprintLibrary.h | The order-building and reading nodes. |
| `ULockstepSimSettings` | LockstepSimSettings.h | Project Settings > Plugins > Lockstep Sim. `Get()`, `ApplyToSession`, `MakeLinkSettings(Role)`. |
| `ILockstepMatchSource`, `FLockstepMatchOptions` | LockstepMatchSource.h | What the host / join screen needs from a game. The component implements it; implement it yourself if your game owns its session directly. |

## Host / join screen

Module **LockstepUI**.

| Type | Header | What it is |
|---|---|---|
| `ULockstepMatchScreenComponent` ("Lockstep Match Screen") | LockstepMatchScreenComponent.h | Add it next to a Lockstep Sim component: shows the screen (Show On Begin Play), Toggle Key, placement (Horizontal / Vertical Alignment, Screen Padding). Nodes: Show, Hide, Toggle, Is Shown. C++: `SetSource` for a game without the component, `GetScreen()`. |
| `ULockstepMatchWidget` (UMG palette "Lockstep") | LockstepMatchWidget.h | The same screen inside your own Widget Blueprint. Call **Set Sim** with your Lockstep Sim component; **On Close Clicked** when Show Close Button is on. |
| `SLockstepMatchScreen` | SLockstepMatchScreen.h | The Slate screen: your name, Host (port, Allow other computers), Join (address), Leave, Repair, status, players panel. `PressHost`, `PressJoin`, `PressLeave`, `PressRepair`, `SetName`, `SetAddress`, `SetPort`, `GetStatusText`. |
| `SLockstepPlayersPanel` | SLockstepMatchScreen.h | The players table alone (number, name, host / you, ping, state). `GetRowsText()`, `CollectPlayers()`. |

## Starter template

Module **LockstepStarter** (editor only: never built into a packaged game); copied into your project by
`Extras/StarterTemplate/CopyStarterToProject.ps1` (see the [Quick start](QuickStart.md)).

| Type | What it is |
|---|---|
| `FStarterRules`, `FStarterUnit`, `FStarterGame`, `EStarterOrder`, `EStarterEvent` | The rules: units with an owner and a target; orders Spawn Unit (1) and Move Units (2); events Unit Spawned and Unit Arrived. |
| `UStarterSimComponent` ("Starter Sim") | The simulation component for those rules, with named orders and the nodes Spawn Unit At, Move Units To, Get Unit Count, Get Unit. |
| `AStarterWorld` | The level's actor: the Starter Sim, a Lockstep Match Screen (key M), the camera, and the drawing. |
| `AStarterPlayerController`, `AStarterGameMode` | Mouse orders, and the game mode (it adds a Starter World to any level that has none). |

## Testing and the proof

| Type | Header | What it is |
|---|---|---|
| `FLockstepTestGroup`, `LOCKSTEP_CHECKF` | LockstepTestKit.h | A simple check group that logs PASS / FAIL (the plugin's own tests use it; Unreal automation tests work just as well, see the starter's tests). |
| `FLockstepTestRecord` | LockstepTestKit.h | Records group results and named checksums for the proof: `AddValue(Name, Value)`, `ExpectGroups(Count)`. |
| `FLockstepProof` | LockstepProof.h | The cross-machine proof: `IsRequested()`, `AddWorkload(Name, Function)`. See [The proof tool](ProofTool.md). |
| `LOCKSTEP_PROOF_WORKLOAD(Id, "Name") { ... }` | LockstepProof.h | Defines a proof workload in any .cpp file, no module class needed (through `FLockstepProofWorkloadRegistrar`). |
| `LockstepSelfTestsRequested()` | LockstepSim.h | Whether the plugin's startup self-tests run this time. |

## Blueprint nodes

### Lockstep Sim component

| Node | What it does |
|---|---|
| **Start Simulation** | Starts (or restarts) the simulation from the component's game. |
| **Issue Order** | Sends an order; returns the tick it runs on, or -1 with an Error that says what is wrong and what to do. |
| **Make Named Lockstep Order** | A new order picked by name from a drop-down ("Spawn Unit"); add its values with the Add nodes. |
| **Get Order Names** / **Get Order Name** / **Find Order Definition** | The known order names; the name of a type number; the definition behind a name. |
| **Is Running**, **Get Tick**, **Get Ticks Per Second**, **Get Input Delay Ticks**, **Get Interpolation Alpha** | The simulation's clock, for drawing between ticks. |
| **Get Last Checksum**, **Is In Sync** | The newest checksum (hex) and its tick; whether everything agrees. |
| **Find State Value**, **Get State Int**, **Get State Paths** | Read any value of the simulation by its name in your `DescribeState` ("unit 3 / position"). |
| **Host Match**, **Join Match**, **Leave Match** | Networking without the screen. |
| **Is In Network Match**, **Is Network Paused**, **Get Local Player**, **Get Player Count**, **Get Network Status** | The match as it is now. |
| **Get Players** | One row per player: name, host, you, waiting, out of sync, repairing, ping. |
| **Repair Match** | Repairs a desync without leaving the match. |
| **Save To File**, **Load From File** | Saves (relative paths go under `Saved/Lockstep/`). |
| **Start Recording**, **Is Recording**, **Stop Recording** | Replays; Stop Recording also checks the replay plays back identically. |

### Lockstep orders and reading (library)

| Node | What it does |
|---|---|
| **Make Lockstep Order** | An empty order of a type number. |
| **Make Lockstep Order From Definition** | An empty order of a definition's type, checked against it when issued. |
| **Add Int**, **Add Int64**, **Add Bool** | Append a value. |
| **Add Position** | A position in centimetres (rounded to whole cm; your C++ reads an `FSimPos`). |
| **Add Fixed** | An exact fraction Numerator / Denominator (`FSimFixed`). |
| **Add Distance** | A distance in centimetres (`FSimFixed` metres). |
| **Validate Order** | Does the order match its definition? |
| **Make Order Reader**, **Make Event Reader** | Read an executed order's or an event's values. |
| **Read Int**, **Read Int64**, **Read Bool**, **Read Position**, **Read Fixed** | Read the next value. |
| **Is Reader OK**, **Is Reader Complete** | No damage / everything read. |
| **Quantize Position** | Where the simulation will put a position (whole centimetres), for previews. |

### Lockstep Match Screen component and widget

**Show**, **Hide**, **Toggle**, **Is Shown** (component); **Set Sim**, **Get Sim** (widget).

### Starter Sim component (the template)

**Spawn Unit At**, **Move Units To**, **Get Unit Count**, **Get Unit**.

## Blueprint events

On the Lockstep Sim component:

| Event | When |
|---|---|
| **On Tick** | After every tick, with the tick that ran. |
| **On Order Executed** | Every order that ran (with its tick and player), in the order the simulation applied them. |
| **On Sim Event** | Every event your rules reported. |
| **On Desync** | The first desync found, with the finder's one-line summary. |
| **On Match Started**, **On Match Ended**, **On Match Paused**, **On Match Resumed** | The match's life. |
| **On Player Joined**, **On Player Left**, **On Player Repaired** | Players. |
| **On Input Delay Changed** | The delay changed (adaptive delay, or the match ended). |

On the Lockstep Match Screen widget: **On Close Clicked**.

## Command line

Read by the Lockstep Sim component (with Accept Command Line on) and the Playground:

| Switch | Effect |
|---|---|
| `-LockstepHost` | Host a match when the simulation starts. |
| `-LockstepJoin` / `-LockstepJoin=<address>` | Join a match (default address: 127.0.0.1). |
| `-LockstepPort=<port>` | The TCP port (default 7777). |
| `-LockstepName=<name>` | The player name. |
| `-LockstepFakeLag=<ms>`, `-LockstepFakeJitter=<ms>` | Test lag on everything this game sends. |
| `-LockstepLogStatus=<seconds>` | Writes the sync / network status and the players list (`Status: PLAYERS: ...`: names, host, you, ping, sync state) to the log every so often, and the checksum after every Nth tick (N = that many seconds of ticks): `Checksum after tick 399: 0x...`, comparable between machines' logs (for runs with `-nullrhi`). |
| `-LockstepLogOrders` | Writes every order handed to the rules: tick, name, its values (as the order script writes them), order type (the number from your order enum), player. Your rules decide what it does. |
| `-LockstepLogState=<filter>` | With each status write (every 5 s unless `-LockstepLogStatus` says otherwise), the state fields whose names contain the filter, with their values. Put a filter with spaces in quotes (`-LockstepLogState="unit 0 /"`). A joining game starts once it has the host's game. |
| `-LockstepOrderScript=<file>` | Gives orders from a text file: lines of `<seconds> <order name> <values>` (positions as X Y Z in cm, whole numbers, `true` / `false`, fractions as `3/4` or `2.5`; `#` comments). With `-LockstepHost` / `-LockstepJoin` the seconds count from the start of the match. A file with a bad line gives no orders (the log names the line and what to do). A relative path starts at the project folder. |
| `-LockstepQuitAfter=<seconds>` | Quits after that many seconds of play (loading does not count), saying goodbye to the other players first. |
| `-LockstepSelfTest` | Run the plugin's self-tests at startup. |
| `-LockstepProof`, `-LockstepProofOut=<file>` | Run the cross-machine proof and quit. |

## Console commands

| Command | Effect |
|---|---|
| `Lockstep.Order <order name> <values>` | Issues a named order on the world's running Lockstep Sim component, e.g. `Lockstep.Order Spawn Unit 2500 4000 0` (positions as X Y Z in cm). The log says the tick it runs on, or what is wrong. |
| `Lockstep.State <filter>` | Writes the state fields whose names contain the filter (empty = all, at most 200) to the log, e.g. `Lockstep.State unit 0 /`. |
| `LockstepSim.Overlay 1` / `0` / `-1` | The on-screen sync overlay on / off / as the settings say. |

## Project settings

**Edit > Project Settings > Plugins > Lockstep Sim** (saved in `Config/DefaultGame.ini`):

| Setting | Default | Meaning |
|---|---|---|
| Ticks Per Second | 20 | The simulation's rate. Every machine of a match, every save and every replay must share it. |
| Input Delay Ticks | 3 | Ticks between giving an order and it running (the host's value is used by everyone). |
| Checksum Interval Ticks | 1 | Compare checksums every N ticks. |
| Run Second Copy | on | A second local copy that catches nondeterminism on one machine. Turn off in a shipped game. |
| Adaptive Input Delay, Adaptive Max Delay Ticks | on, 20 | The host raises / lowers the delay to the players' connections. |
| Port, Bind Address | 7777, 127.0.0.1 | Where a host listens (0.0.0.0 = other computers too). |
| Max Players | 8 | 2 to 8. |
| Fake Lag Ms, Fake Jitter Ms | 0, 0 | Test lag. |
| Pause Timeout Seconds | 60 | How long a lost player is waited for. |
| Silence Timeout Seconds | 5 | Silence that counts as a lost connection. |
| Show Overlay | off | The on-screen sync lines. |
| Run Self Tests At Startup | off | The plugin's self-tests every time the game starts. |
