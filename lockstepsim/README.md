# Lockstep Sim documentation

The documentation for **Lockstep Sim**, an Unreal Engine plugin by Grayback Games, sold on Fab. The same pages ship inside the plugin's `Docs` folder.

Deterministic lockstep simulation for Unreal Engine: thousands of units in multiplayer, with only the
players' orders on the network. Fixed-point math, a fixed tick, checksums and a desync finder that names
the field that went wrong, up to 8 players with join-in-progress, pause and resume, repair without
leaving the match, saves, replays, a host / join screen, Blueprint support, and a starter template.

## Start here

1. [Quick start](QuickStart.md): from the starter template to a running two-player match.
2. [Core concepts](Concepts.md): determinism, ticks, orders, input delay, checksums, desyncs, repair,
   replays, in plain language.
3. [Adding your own orders and rules](OrdersAndRules.md): a walk-through, plus the determinism rules.
4. [Reference](Reference.md): the C++ API and every Blueprint node.
5. [Troubleshooting](Troubleshooting.md): finding and fixing desyncs, connecting, stutter.
6. [The proof tool](ProofTool.md): check that two computers run your game identically.
7. [Changelog](CHANGELOG.md)

## What is in the plugin

| Module | What it is |
|---|---|
| **LockstepSim** | The framework: fixed-point math, positions, the world and session, orders, networking, saves, replays, the desync finder, Blueprint support, project settings, the overlay. |
| **LockstepUI** | The host / join screen and the players panel (Slate, a UMG widget and an actor component). |
| **LockstepStarter** | The starter template: a minimal multiplayer game to copy into your project. Map `/LockstepSim/Starter/L_Starter`. Editor only (never in a packaged game). |
| **LockstepPlayground** | A hands-on demo with a guided tutorial: thousands of units, sliders for lag and input delay, buttons that cause and repair desyncs. Map `/LockstepSim/Playground/L_Playground`. Editor only (never in a packaged game). |

## Requirements

- Unreal Engine 5.8, Windows (64-bit).
- A C++ project (the rules of a lockstep game are C++; Blueprints issue orders, read state and react to
  events). A Blueprint-only project can be converted by adding any C++ class (Tools > New C++ Class).

## What this plugin does not include

So you know before you build on it:

- **No matchmaking, lobby service or relay.** Players join by typing the host's address as numbers
  (IPv4, like `192.168.1.20`, or `localhost`). Names are not looked up. To play over the internet the
  host needs a port forward for the TCP port.
- **No encryption and no password.** Connections are plain TCP. Anyone who can reach the host's port
  with the same build and rules can join, up to the player limit. A host listens on its own PC only
  until **Allow other computers** is switched on.
- **No cheat protection.** In lockstep every player's computer holds the whole game, so a modified
  game could read information your game hides, such as units under fog of war. This suits co-op and
  games among friends. A competitive game needs its own measures.

## Support

Questions, bug reports and feedback: [graybackgames@gmail.com](mailto:graybackgames@gmail.com).
