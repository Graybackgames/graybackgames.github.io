# Adding your own orders and rules

Your game's rules are one C++ class that implements `ILockstepGame`. This page walks through adding a new
order to the starter template, then covers the rules every piece of simulation code must follow.

Everything here uses the starter's names (`FStarterGame`, `UStarterSimComponent`). After you copy the
template into your project and rename it (see the [Quick start](QuickStart.md)), use your own names.

## The pieces

| Piece | Where (starter) | What it does |
|---|---|---|
| Rules numbers | `FStarterRules` | Speeds, limits, sizes. Written per second, converted once for the tick rate. Hashed into every checksum. |
| State | `FStarterGame` members (`Units`) | Everything that changes while the game runs. Plain data in arrays. |
| Order types | `EStarterOrder` | The numbers that travel between machines. Never renumber one. |
| Order builders | `FStarterGame::MakeSpawnUnit` ... | Write an order's values (its payload) in a fixed order. |
| `ApplyOrder` | `FStarterGame::ApplyOrder` | Reads an order's values back in the same order and changes the state. |
| `Tick` | `FStarterGame::Tick` | Advances the rules by exactly one tick. |
| `AddStateToChecksum`, `SaveState`, `LoadState`, `DescribeState` | `FStarterGame` | The same fields, in the same order, four times: hashing, saving, loading, naming for the desync finder. |
| Order names | `UStarterSimComponent::RegisterOrderNames` | Names and value lists for Blueprints (Make Named Lockstep Order). |
| Issuing | `UStarterSimComponent::SpawnUnitAt`, or any Blueprint | Turns Unreal values into an order and sends it. |

## Example: a "Stop Units" order

Goal: a key that makes all of the player's units stop where they are.

### 1. Give the order a number

```cpp
enum class EStarterOrder : uint16
{
	SpawnUnit = 1,
	MoveUnits = 2,
	StopUnits = 3,   // new: no values
};
```

Numbers are part of the network protocol and of every save and replay. Add new ones at the end; never
change or reuse an old one.

### 2. Write a builder

```cpp
// StarterGame.h
static FLockstepOrder MakeStopUnits();

// StarterGame.cpp
FLockstepOrder FStarterGame::MakeStopUnits()
{
	FLockstepOrder Order;
	Order.Type = uint16(EStarterOrder::StopUnits);
	return Order;   // no payload
}
```

An order with values writes them with `FLockstepOrderWriter`: `AddInt32`, `AddInt64`, `AddBool`,
`AddFixed`, `AddPos`. Keep the writer in a named variable and call it once per value (the `Add` calls
return nothing, so they cannot be chained). For a single value a temporary writer on one line is fine; the
template's own builders do that: `FLockstepOrderWriter(Order).AddPos(To);`. A builder for an order with two values, say a "Shift Units"
order with an X and a Y offset in metres (in the starter's top-down view +X is up on the screen and +Y is to
the right):

```cpp
FLockstepOrder FStarterGame::MakeShiftUnits(int32 XMeters, int32 YMeters)
{
	FLockstepOrder Order;
	Order.Type = uint16(EStarterOrder::ShiftUnits);
	FLockstepOrderWriter W(Order);
	W.AddInt32(XMeters);
	W.AddInt32(YMeters);
	return Order;
}
```

`ApplyOrder` then reads them back in the same order (`R.ReadInt32()`, then `R.ReadInt32()`). Each `Add`
call has its `Read`: `ReadInt32`, `ReadInt64`, `ReadUInt8`, `ReadBool`, `ReadFixed`, `ReadPos`. The player
who gives the order is stamped automatically (`Order.Player`); never put it in the payload.

A value that names a player (for example "give this unit to player N") uses the code's numbers, which start
at **0**, the same as `Order.Player` and `GetLocalPlayer()`. Logs, status lines and state lines count from
**1**, so the same player reads one higher there (see [Player numbers](Concepts.md#player-numbers)). Name such
a parameter so it says which numbering it takes, for example `ToPlayerIndex`.

### 3. Apply it

In `ApplyOrder`, read the values back **in the same order** with `FLockstepOrderReader` (the starter's
`ApplyOrder` starts with `FLockstepOrderReader R(Order);`, and `Context` is its tick context: the examples
below use both), check `IsComplete()` (a damaged order must be ignored, never half applied), then change
the state:

```cpp
case EStarterOrder::StopUnits:
	for (FStarterUnit& U : Units)
	{
		if (U.Owner == Order.Player) { U.Target = U.Pos; U.bMoving = false; }
	}
	break;
```

Each `case` reads its own order's values (the starter's Spawn Unit and Move Units each read a position
first); Stop Units has none, so it reads nothing.

An order that needs randomness takes it from `Context.Rng`, the simulation's own generator, which advances
the same way on every machine. Use it only inside `ApplyOrder` and `Tick`, and visit objects in id order so
every machine draws the numbers for the same objects. A "Scatter Units" order that sends each unit up to
`Radius` metres away in a random direction (with `ScatterUnits = 4` added to `EStarterOrder` and a builder that writes the radius with `AddInt32`):

```cpp
case EStarterOrder::ScatterUnits:
{
	const int32 Radius = R.ReadInt32();
	if (!R.IsComplete() || Radius < 1 || Radius > 50) { return; }
	const FSimFixed Max = FSimFixed::FromInt(Radius);
	for (FStarterUnit& U : Units)   // id order
	{
		if (U.Owner != Order.Player) { continue; }
		FSimPos Spot = U.Pos;
		Spot.Offset(Context.Rng.RangeFixed(-Max, Max), Context.Rng.RangeFixed(-Max, Max));   // both ends included
		U.Target = ClampToArena(Spot);
		U.bMoving = U.Target != U.Pos;
	}
	break;
}
```

### 4. Name it for Blueprints (optional)

```cpp
void UStarterSimComponent::RegisterOrderNames()
{
	// ... existing orders ...
	RegisterOrder(TEXT("Stop Units"), int32(EStarterOrder::StopUnits), {},
		NSLOCTEXT("Starter", "StopUnits", "Stops all of the ordering player's units where they are."));
}
```

Blueprints can now use **Make Named Lockstep Order** with "Stop Units" picked from the drop-down, then
**Issue Order** on the component. "Stop Units" has no values, so its list is empty (`{}`). An order with values
lists them in the order its builder writes them, separated by commas:

```cpp
RegisterOrder(TEXT("Send Squad"), int32(EStarterOrder::SendSquad),
	{ Param(TEXT("Where"), ELockstepParamKind::Position), Param(TEXT("Count"), ELockstepParamKind::Int) },
	NSLOCTEXT("Starter", "SendSquad", "Sends the ordering player's first Count units to Where."));
```

In Blueprints each value must then be added with the matching Add node (Add Position, then Add Int), in that
order; Issue Order refuses the order with a message if not.

### 5. Issue it

Give the component a function, next to `SpawnUnitAt`:

```cpp
// StarterSimComponent.h
// Stops all of your units where they are. Returns the tick it runs on, or -1.
UFUNCTION(BlueprintCallable, Category = "Starter")
int32 StopUnits();

// StarterSimComponent.cpp
int32 UStarterSimComponent::StopUnits()
{
	if (!GetSession()) { return -1; }   // the simulation is not running yet
	return int32(GetSession()->QueueOrder(FStarterGame::MakeStopUnits()));
}
```

`QueueOrder` returns the tick the order will run on (now + the input delay). Then bind a key to it in
`AStarterPlayerController::SetupInputComponent`, like the mouse buttons there:

```cpp
InputComponent->BindKey(EKeys::S, IE_Pressed, this, &AStarterPlayerController::StopUnits);

void AStarterPlayerController::StopUnits()   // declared in StarterWorld.h, next to MoveUnits
{
	if (AStarterWorld* World = FindWorld()) { World->Sim->StopUnits(); }
}
```

and add the key to the controls hint in `AStarterWorld::BeginPlay`.

For an order with a value chosen by the key (say keys 1 to 4 pick a corner), bind every key to one function
that takes the key (`BindKey` has a form whose function receives the `FKey`), and turn the key into the value
(the component's `CornerUnits(int32 Corner)` is written like `StopUnits` above, with the value passed to its builder):

```cpp
InputComponent->BindKey(EKeys::One, IE_Pressed, this, &AStarterPlayerController::CornerUnits);
InputComponent->BindKey(EKeys::Two, IE_Pressed, this, &AStarterPlayerController::CornerUnits);
InputComponent->BindKey(EKeys::Three, IE_Pressed, this, &AStarterPlayerController::CornerUnits);
InputComponent->BindKey(EKeys::Four, IE_Pressed, this, &AStarterPlayerController::CornerUnits);

void AStarterPlayerController::CornerUnits(FKey Key)   // declared in StarterWorld.h
{
	const int32 Corner = Key == EKeys::One ? 0 : Key == EKeys::Two ? 1 : Key == EKeys::Three ? 2 : 3;
	if (AStarterWorld* World = FindWorld()) { World->Sim->CornerUnits(Corner); }
}
```

In actor or controller code, do not name a local variable or parameter after a member the class already has,
such as `Owner` (`AActor::Owner`): the compiler stops with error C4458 ("declaration hides class member").
The template uses `OwnerPlayer` for this reason; pick a name like that.

Without a key (a script, a test, a game with no screen), a named order can also be given as text:
`-LockstepOrderScript=<file>` on the command line, or the console command `Lockstep.Order Stop Units` (see
the [Quick start](QuickStart.md#with-no-screen-a-script-a-build-server)).

A player number in an order's values is written the code's way there too: a script line that gives a unit
"to player 0" (`5 Give Unit 3 0`: after 5 s, unit 3, `ToPlayerIndex` 0) gives it to the host, and the log
and state lines show that player as player 1.

### 6. Test it

Add a test to your `<Prefix>GameTests.cpp` (copy the `MakeScript` / `RunScript` pattern there) that checks
two things. Stamp each order with `WithPlayer(Order, Player, Tick, Sequence)`: `Sequence` decides the order in
which one player's orders run within a tick, so give each player a count that rises with every order it gives
(the script's `Sequence[Player]++`), whatever the tick. To add an order to the result of `MakeScript`, use any
`Sequence` above the script's (it stays below 100, so 100, 101, ... are fine). The comment above `MakeScript`
gives its timeline (player 2's units walk until tick 269, player 1's until tick 412), so you know which units
are still walking when your order runs.

Unit ids, for an order that names units: a unit's id is its place in the unit list, given in the order the
units spawn (the first unit is 0). Within one tick, orders run by player (player 1's first), then by
`Sequence`. So in `MakeScript`, where player 1 spawns on ticks 2-7 and player 2 on ticks 3-8, player 1 owns
units 0, 1, 3, 5, 7, 9 and player 2 owns 2, 4, 6, 8, 10, 11. A test can also read each unit's owner from the
state (`GetUnits()`, `Owner`) instead of counting.

1. **It is deterministic.** Run a script with the new order twice (same seed, same orders): the checksums
   are equal on every tick. Leave the order out: they differ from its tick on.
2. **It does what it should.** Read the state after the order and check the effect: for Stop Units, every
   unit of the ordering player has stopped where it was, and the other player's units still walk. For an
   order that sends units to a computed spot, work out the expected spot with the same helper the rules use
   (`Game.ClampToArena(...)` is public for that). The
   determinism checks alone pass for a wrong rule too, as long as it is wrong the same way everywhere.
   (The starter's tests do both: see "every unit reached its spot and stopped".)

Then break the rule on purpose once and see the test fail. Also add the order to the proof workload at the
end of the same file (the `LOCKSTEP_PROOF_WORKLOAD` block), so the [proof tool](ProofTool.md) compares it
between computers, and check that `checksum.<Prefix>RulesAllTicks` in the results changed (the checksums at
ticks 1000 / 2000 / 3000 may not, if a later order overwrites your order's effect). Compare with a run made
after your rule and state changes, just before you add the order to the workload: a new hashed field changes
every value by itself, so only that comparison shows the workload really gives the order. Run the tests: **Tools > Session Frontend**, the
**Automation** tab, filter `<Prefix>.Rules` (see the [Quick start](QuickStart.md), step 6, for the command line).

A new order needs no other changes: it is saved, loaded, replayed, sent and checked like every other.

## Adding state

When you add a field (say, `int32 Health` on a unit):

1. Give it a default value.
2. Hash it in `AddStateToChecksum`.
3. Write it in `SaveUnit` (the starter's helper that `SaveState` uses for each unit) and read it in
   `LoadState`, at the same position. `LoadState`'s size check takes the size of a saved unit from
   `SaveUnit` itself, so it needs no change.
4. List it in `DescribeState` (`D.AddInt(Name + TEXT("health"), U.Health)`), at the same position.

Keep the four functions in the same order. If a field is hashed but not described, the desync finder
says "every field the game describes is equal, yet the checksums differ" instead of naming it.

Saves from before the change will no longer load (their layout is different), which is expected while
you develop.

## Adding a rules number

Put it in your rules struct, written in real units, converted once:

```cpp
FSimFixed RunStep;          // in the constructor:
RunStep = FSimPerSecond::PerTick(8, 1, TicksPerSecond);   // 8 m/s
```

Then add it to `AddToChecksum` and `Describe`. Two machines with different values are then caught on the
first tick, and the report names the rule.

Rates of whole things (1 wood per 2 seconds) use `FSimRate::PerSeconds(1, 2, TicksPerSecond)` with an
`int32` progress counter in the state (hashed and saved like any field).

## Your own position math

`Offset` and `StepToward` cover moving by a distance. For anything else (a point part of the way to another,
a mirrored or rotated spot), work in whole numbers: `WorldRawX()` / `WorldRawY()` give a position as `int64`
in raw units (1/65536 m, so 1 m = 65536), `FSimPos::DeltaRawX(From, To)` / `DeltaRawY` the difference, and
`FSimFixed::FloorDiv(A, B)` divides `int64` rounding down (the same on every machine, also for negative numbers).
Build the result with `FSimPos::FromWorldRaw(X, Y)`. For example, a point Percent of the way from A to B:

```cpp
const int64 X = A.WorldRawX() + FSimFixed::FloorDiv(FSimPos::DeltaRawX(A, B) * Percent, 100);
const int64 Y = A.WorldRawY() + FSimFixed::FloorDiv(FSimPos::DeltaRawY(A, B) * Percent, 100);
const FSimPos Spot = ClampToArena(FSimPos::FromWorldRaw(X, Y));
```

To scale by an `FSimFixed` instead (a factor such as 1.5 from an order), multiply by its raw value and divide
by `FSimFixed::One64` (65536, the raw value of 1.0); `Factor.Raw` is the value in 1/65536 steps. For a point
Factor times as far from a centre C as P is:

```cpp
const int64 X = C.WorldRawX() + FSimFixed::FloorDiv(FSimPos::DeltaRawX(C, P) * Factor.Raw, FSimFixed::One64);
const int64 Y = C.WorldRawY() + FSimFixed::FloorDiv(FSimPos::DeltaRawY(C, P) * Factor.Raw, FSimFixed::One64);
```

(The product fits in `int64` while the distance in metres times the factor stays under about 2 billion: a
1,000 km map with factors up to 128 is far inside that.)

For directions, use `FSimVec2` (add `#include "SimMath.h"` to your `<Prefix>Game.cpp`; the template does not
include it). Make one from two positions with their raw difference: an
`FSimFixed` holds about ±32 km, so `FSimFixed::Saturate` keeps a longer difference at that limit (same
direction only along an axis; keep vectors to distances under 32 km). For example, a unit P walks Meters
straight away from a centre C:

```cpp
const FSimVec2 Away(FSimFixed::FromRaw(FSimFixed::Saturate(FSimPos::DeltaRawX(C, P))),
	FSimFixed::FromRaw(FSimFixed::Saturate(FSimPos::DeltaRawY(C, P))));
const FSimVec2 Step = Away.ScaledToLength(FSimFixed::FromInt(Meters));   // stays zero when P is on C
FSimPos Spot = P;
Spot.Offset(Step.X, Step.Y);
U.Target = ClampToArena(Spot);
```

## The determinism rules

Everything inside your `ILockstepGame` (`ApplyOrder`, `Tick`, and anything they call) must follow these
rules. Breaking one compiles fine and desyncs later, often only on some machines.

1. **No `float` or `double`** in state or math. Use `FSimFixed` (numbers), `FSimPos` (positions),
   `FSimAngle`, `FSimMath`, `FSimTrig`, `FSimVec2`. Floats are only for drawing, never fed back.
2. **Random numbers only from `Context.Rng`** (`RangeInt`, `NextUnit`, `RangeFixed`), only inside the tick.
   Never `FMath::Rand`, `FRandomStream` or seeds from the clock.
3. **No iteration over `TMap` / `TSet`** where order matters. Keep objects in arrays with stable integer ids;
   process them in id order. Break ties by id. Sorting an array (`TArray::Sort`) is fine when the comparison
   breaks every tie, for example by id, so no two elements ever compare equal: then every machine gets the same
   order, even though the sort itself is not stable.
4. **Read nothing from outside**: no clock, frame time, physics, navigation, animation, actors or engine
   queries. The only inputs are the previous state, the tick number and the orders for that tick.
5. **The tick is fixed.** Never advance by a variable amount. Rates written per second are converted once.
6. **No pointers or memory addresses** as keys, sort keys or checksum input.
7. **Integer math with known rounding.** `FSimFixed` floors and saturates by definition; if you write your
   own integer math, use `int64` for intermediate results and do not shift negative numbers left.
8. **Hash all state** in `AddStateToChecksum`, field by field in a fixed order (never raw struct memory:
   padding bytes are random).
9. **Test every new rule**: same seed + same orders = same checksum, twice.

## Converting from Unreal values

Unreal positions are floating-point centimetres. Convert them **once, when issuing an order**, with
`FLockstepQuantize::CmToSimPos` (C++) or the Blueprint node **Add Position (cm)**: the order carries the
exact sim value as bytes, so every machine gets the same number. The way back, for drawing, is
`FLockstepQuantize::SimPosToCm`.

## Drawing

Read the state through `GetSession()->GetWorld().GetGame<FYourGame>()` (C++) or the component's
**Find State Value** / **Get State Int** (Blueprints, by the names in `DescribeState`). Draw between ticks
with `GetInterpolationAlpha()`. React to things that happened with **events**: the rules call
`Context.Emit(Type)` and write a payload; the component fires **On Sim Event** after the tick (the starter
reports `UnitSpawned` and `UnitArrived`). Events are output only: they never change the simulation.
