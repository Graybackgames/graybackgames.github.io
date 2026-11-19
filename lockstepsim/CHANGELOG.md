# Changelog

## 1.0.0 (first release)

The framework
- Fixed-point math (`FSimFixed`, Q16.16), positions on maps of any size (`FSimPos`, 256 m chunks with
  64-bit distances), angles, sine / cosine / arctangent from lookup tables, 2D vectors, whole-unit rates,
  per-second conversions, a seeded random generator (PCG32) and an FNV-1a checksum. No floating point
  anywhere in the simulation.
- A fixed-tick world (`FLockstepWorld`, 1 to 120 ticks per second, 20 by default) running your rules
  through one interface (`ILockstepGame`), with a rules fingerprint in every checksum.
- Sessions (`FLockstepSession`): the order queue with input delay, real-time pacing, a second local copy
  that catches nondeterminism on one machine, adjustable checksum interval.
- The desync finder: when copies or machines disagree, a plain-language report of the first tick, object
  and field that differ, swapped automatically between machines and written to `Saved/Desync/`.
- Saves (exact, refused if the rules differ) and replays (every tick re-checked on playback).

Networking
- Up to 8 players over TCP: one host, players joining at any time (also while the match runs), every
  bundle relayed by the host.
- Adaptive input delay: the host measures every connection and changes the delay for everyone on the
  same tick.
- A lost connection pauses the match and resumes on the same tick when the player is back; a player
  cut off while joining keeps their seat.
- Repair inside the match: an out-of-sync player gets a fresh copy of the host's world without anyone
  leaving; orders given meanwhile are kept.
- Player names, and a players list with pings and sync state on every machine.
- Fake lag and jitter for testing on one PC.

Unreal integration
- The Lockstep Sim component: run a simulation in any actor; Blueprint nodes to build and issue orders
  (also by name, picked from a drop-down), read the simulation's state by name, and react to ticks,
  executed orders, game events, desyncs and network changes; host, join, save, load and record replays.
- The host / join screen and players panel (Lockstep Match Screen component, UMG widget, Slate).
- Project settings (Project Settings > Plugins > Lockstep Sim) and an on-screen sync overlay.
- For scripts and build servers: orders from a text file (`-LockstepOrderScript`) or the console
  (`Lockstep.Order`); in the log: every order handed to the rules, per-tick checksums comparable between
  machines, the players list, chosen state fields (`-LockstepLogState`, `Lockstep.State`), and status lines;
  quitting after a set time of play.

Examples and tools
- The starter template: a minimal multiplayer game, copied into your project (renamed, with its modules, the plugin
  and its game mode set up) by one script.
- The Playground: thousands of units, every network condition on sliders, a guided eight-lesson tutorial.
- Both examples load in the editor only and are never built into a packaged game.
- The cross-machine proof tool (your own workloads from any .cpp file with `LOCKSTEP_PROOF_WORKLOAD`), and the
  plugin's self-tests.
- Documentation: quick start, concepts, guide, reference, troubleshooting.
