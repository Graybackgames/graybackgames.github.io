# Core concepts

This page explains the ideas behind Lockstep Sim in plain terms. You do not need any of it to run the
starter template, but you will need all of it the first time something goes wrong.

## Lockstep in one paragraph

Most Unreal multiplayer games send the *state* of the world over the network: positions, health,
animations. That works for a few dozen characters, but not for thousands of units. Lockstep sends only
the *orders* players give ("move these units there"). Every machine runs the same simulation, applies
the same orders at the same moment, and so computes the same world. A thousand units cost the network
nothing extra; only the orders travel.

The catch: every machine must compute **exactly** the same result, bit for bit, forever. That is what
the plugin is for.

## Determinism

A simulation is *deterministic* when the same starting state and the same orders always give the same
result, on every computer. Unreal's normal tools are not deterministic in this strict sense:

- **Floating-point math** (`float`, `double`) can round differently on different CPUs, compilers and
  settings. Lockstep Sim uses fixed-point numbers instead: `FSimFixed` (whole numbers underneath, with
  1/65536 precision) and `FSimPos` (positions on maps of any size).
- **Physics, navigation, animation and timers** depend on frame time and are not reproducible. The
  simulation never reads them.
- **Random numbers** must come from the simulation's own generator (`Context.Rng`), which every machine
  seeds and advances the same way.
- **Iteration order** of maps and sets (`TMap`, `TSet`) is not guaranteed. Keep simulation objects in
  arrays and process them by index.

Your game's rules live in a C++ class that implements `ILockstepGame`. Everything inside it follows these
rules. Everything outside it (drawing, sound, UI, cameras) can use Unreal normally, as long as it only
*reads* the simulation and never writes to it.

## Ticks

The simulation advances in fixed steps called **ticks**, 20 per second by default (Project Settings >
Plugins > Lockstep Sim > Ticks Per Second). A tick is always the same amount of game time, whatever the
frame rate. Drawing happens between ticks: `GetInterpolationAlpha()` tells you how far real time is
between the last tick and the next, so units move smoothly even at 144 fps.

Write speeds and rates *per second* and convert them once for the tick rate (`FSimPerSecond::PerTick`,
`FSimRate::PerSecond`), and your game plays at the same speed at any tick rate.

## Orders

An **order** is the only way anything gets into the simulation: a small packet with a type number (your
game decides what 1, 2, 3 mean) and a payload of values (positions, ids, numbers). The player who gives it
is stamped on it automatically.

You can give orders names ("Spawn Unit", "Move Units") so Blueprints can pick them from a drop-down. See
[Adding your own orders and rules](OrdersAndRules.md).

Values from Unreal (a mouse position in centimetres, a float slider) are converted to simulation values
**once, on the machine that gives the order**. The order then carries exact bytes, so every machine gets
identical values whatever its floating point does.

## Input delay

An order given now does not run now: it runs a few ticks later, on every machine at once. That gap is the
**input delay** (3 ticks = 150 ms by default). It gives the order time to reach the other players before
its tick comes up.

- Too little delay for the connection: the game has to wait for late orders and stutters.
- Too much: the game feels sluggish.

With **adaptive input delay** (on by default) the host measures every player's connection and raises or
lowers the delay for everyone together, so the match runs at full speed. Every machine switches on the
same tick.

When a player controls a unit directly, the plugin's prediction helpers can hide the delay: draw the unit
where it *will* be once the orders in flight have run, and correct smoothly if the simulation disagrees.

## Checksums

After a tick, the simulation can compute a **checksum**: a 64-bit fingerprint of the entire world state.
Machines exchange checksums with their orders. If two machines' checksums ever differ, their worlds have
drifted apart: a **desync**.

Checksums also include a fingerprint of the *rules* (every tuning number, the tick rate), so two players
running different versions or mods are caught on the very first tick.

By default a checksum is computed every tick. For big worlds, Checksum Interval Ticks can make it rarer
(every 20 ticks is once a second); a desync is then noticed on the next checked tick.

While developing, keep **Run Second Copy** on: the plugin runs two copies of the world on the same
machine and compares them every checked tick. Anything that is not deterministic (a stray float, a `TMap`
loop) shows up immediately, without a second computer. Turn it off in a shipped game to halve the cost.

## Desyncs and the desync finder

A desync is always a bug in the rules (or a mismatch of versions). A checksum says *that* two worlds
differ, not *where*. The **desync finder** answers where: your game lists its state as named values
(`DescribeState`: "unit 3 / position"), and when a desync is caught, the plugin compares those lists and
reports, in plain language, the first tick, the object and the field that differ. Between machines, the
two machines swap their lists automatically and both write the report to `Saved/Desync/`.

See [Troubleshooting](Troubleshooting.md) for how to read a report and fix the usual causes.

## Repair

A desynced player can be **repaired** without leaving the match: the host sends that player a fresh copy
of its world, the player carries on from there, and the match never stops. Orders the player gives while
the copy is on its way are kept and run as soon as it arrives. The host's world is always the reference.

Repair is a recovery tool, not a fix: the bug that caused the desync is still there. Use the report to
find it.

## Network matches

One machine **hosts**; up to seven others **join** it (8 players in all). Each joining machine receives a
copy of the host's world, so players can join a match that is already running. The host relays every
player's orders to everyone else.

- Each player's orders for one tick travel together as one **bundle** (an empty one if the player gave no order), so
  every machine knows when it has everything for a tick. The log mentions bundles when it resends them after a lost
  connection.
- A tick runs only once every player's orders for it have arrived: the network sets the pace, and slow
  connections make everyone wait (raise the input delay, or let adaptive delay do it).
- A **lost connection pauses** the match on every machine until the player is back (up to the Pause
  Timeout, 60 s by default); it then carries on from the same tick.
- A player who **leaves** is removed; the others play on.
- **The host leaving ends the match** for everyone; each player keeps their world and plays on alone.

The host / join screen (Lockstep Match Screen) and its players panel show who is connected, their ping
and whether they are in sync.

## Saves and replays

Because the world is plain data, saving is exact: a save holds the whole state (including the random
generator and the orders already given but not yet run) and loading it continues the game exactly as if
it had never stopped. Saves made with different rules are refused with a clear message.

A **replay** is the starting state plus every order that ran, plus the checksum of every tick. Playing it
back re-runs the simulation and checks every tick, so a replay is also a determinism test: if it does
not play back identically, something outside the orders changed the world.

## Player numbers

In C++ and Blueprints, player numbers start at **0** (0 = the host, or the only player when playing
alone). Everything shown on screen and written to the log counts from **1** ("Player 1" is the host;
`GetLocalPlayer()` = 1 is "player 2" in a message). Your own `DescribeState` decides how your state
reads in desync reports and `-LockstepLogState`; the starter adds 1 there too (`unit 0 / player = 2`).

An order value that names a player also uses the code's numbers (from 0), like `Order.Player`: an order
script line that gives a unit "to player 0" gives it to the host, which the log and state lines call
player 1. Name such a parameter so it says so, for example `ToPlayerIndex`.
