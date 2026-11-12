# multiplayer-everything

Multiplayer between two different games, on two machines.

You play Minecraft on your Mac, and a friend plays Terraria on theirs. You see each other, wave, chat and hit each other. Each game draws the other player as one of its own characters. You get one download, a pairing code, and you're in.

```sh
multiplayer host                 # shows a code, e.g. amber-otter-42
multiplayer join amber-otter-42  # on your friend's machine
```

## Why
Running Minecraft inside other games has become a genre, with dozens of projects across dozens of games. But every one of them runs on a single machine and installs like a developer tool. multiplayer-everything makes it multiplayer across machines, and as easy to install as a normal app.

- Every game speaks one small, game-neutral protocol, so each new adapter works with every existing one.
- Covalent agents set up both sides and build adapters for new games.

## Status
This repo is at the spec stage, with no code yet.
- **Hub game:** Minecraft Java.
- **Second game:** Terraria or Stardew Valley, to be decided.

## Read
- [SPEC.md](SPEC.md): what we're building, the protocol, safety, and what counts as done.
- [PLAN.md](PLAN.md): work packages, milestones, time estimates, costs and risks.
