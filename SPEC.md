# multiplayer-everything: spec

## 1. The product
You play Minecraft on your Mac, and a friend plays another game on theirs. You see each other, wave, chat and hit each other. Each game draws the other player as one of its own characters.

**What already exists:**
- **Single-machine mashups:** Minecraft has been run inside dozens of other games, but always on one machine. The two games share a frame, and the install is developer-grade: build tools, compilers, graphics injectors.
- **The Minecraft↔Hytale crossplay mod:** proves cross-game multiplayer is possible, but it's a proof of concept with no download.

**What's new here:**
- multiplayer between two *different* games on two *different* machines;
- one signed download;
- a protocol that lets any adapted game meet any other.

### User flow
1. Download the app and open it. It's notarized, so macOS shows no warning.
2. The app lists your games, for example: "Minecraft Java: ready", "Terraria: ready".
3. Click **Install** next to your game. The adapter goes into a separate game profile, and your own worlds are never touched.
4. One player clicks **Host** and gets a code like `amber-otter-42`. The other clicks **Join** and types it.
5. Both launch their game from the app, and you see each other.

**Target:** download to seeing your friend in **under 5 minutes**.

## 2. Parts

### 2.1 The app (macOS first, Windows in phase 2)
- **Detect:** finds Minecraft launcher folders and Steam libraries.
- **Install:** puts the adapter into a separate game profile, with a backup. Remove restores the game exactly, verified by checksum.
- **Pair:** Host and Join use short codes.
- **Run:** a daemon carries the traffic between your game and the relay. The UI is a local web page the daemon serves.
- **CLI:** `multiplayer install | doctor | host | join | test | remove`. `doctor --json` is machine-readable, so agents can use it.
- **Stack:** TypeScript, compiled to one binary. No Electron.

### 2.2 Adapters
Each adapter is one mod per game, built on that game's official mod loader. It:
- sends the local player's pose 20 times a second;
- draws the friend as a native stand-in: snapped to the local ground, no collision, name tag, chat bubble;
- applies incoming hits, chat and emotes to the local player.

### 2.3 Relay
The relay runs on **Cloudflare Workers with Durable Objects**, which needs no server to run and starts on the free plan.
- A Worker accepts the WebSocket and routes it, using the pairing code, to **one Durable Object per room**.
- The room's Durable Object holds both players' connections and forwards each message to the other player.
- It uses the WebSocket Hibernation API, so time is billed only while a message is being handled.
- Two players per room in v1. Rooms expire after 12 hours.
- It stores nothing beyond the live session.

**Free-plan budget:**
- Cloudflare allows 100,000 requests a day. Incoming WebSocket messages count at 20:1, and outgoing ones are free.
- At 10 relay messages a second per player (see 3, Transport), one room uses about 3,600 requests an hour, so the free plan covers **about 28 hours of play a day**, across all rooms.
- Beyond that, the $5/month Workers Paid plan costs about $0.0005 per extra room-hour.

**When the daily limit is reached:** new requests fail until 00:00 UTC. The app then shows "relay busy, try later" rather than crashing.

**Later optimization:** connect players directly with WebRTC. Cloudflare's STUN service is free and unlimited, and its TURN fallback includes 1,000 GB a month free. Most sessions would then skip the relay.

### 2.4 Multiplayer protocol v1
The protocol is game-neutral: each adapter translates its own game to and from it. N adapters work with each other without N² one-off bridges.

## 3. Protocol v1

### Transport
- **Mod ↔ app:** WebSocket on `127.0.0.1` with a per-session token. The app writes the port and token into the game profile at launch.
- **App ↔ relay:** WSS. Adapters send poses 20 times a second, and the daemon bundles two poses into each relay message, so each player sends 10 relay messages a second. Events are sent immediately. Smoothing (below) hides the bundling.

### Messages (JSON)
| Message | Fields |
|---|---|
| `hello` | `proto: 1`, `game`, `adapterVersion`, `caps[]`, `dims: 2\|3`, `view: side\|top\|3d`, `name`, `look` |
| `pose` | `t`, `p: [x, y, z]` (meters from the anchor), `yaw`, `pitch`, `v: [vx, vy, vz]`, `anim`, `hp` (0–1) |
| `event` | one of `hit {amount}`, `chat {text}` (≤200 chars), `emote {id}`, `respawn` |
| `bye` | (none) |

`anim` is one of `idle`, `walk`, `run`, `jump`, `fall`, `swim`, `fly`, `attack`, `hurt` or `dead`.

### Coordinates
- **Axes:** Y is up and units are meters. x = east and z = south, which are Minecraft's axes, since Minecraft is the hub game.
- **Anchor:** each player's spawn point, or a spot they choose with `/mp anchor`, is zero. Positions are offsets from it.
- **Side-view 2D games** (Terraria): map x and y, send z = 0, and ignore the incoming z.
- **Top-view 2D games** (Stardew Valley): map x and z, ignore height, and show jumps as a hop animation.

### Smoothing
- The receiver draws the stand-in 100 ms behind live.
- It predicts ahead up to 250 ms when packets are late.
- A stand-in that has been silent for 2 seconds fades out.

### Hits
- v1 applies a flat 10% of max HP per hit; it's configurable.
- Sessions are friends-only (joining needs the code), so hits are taken on trust. There's no competitive anti-cheat.

### Limits
- The daemon validates every message against the schema and clamps every number.
- At most 30 messages a second, per player.
- The only free text is chat and player names. Code and file paths never cross the wire.

## 4. Adapters in the MVP

### Minecraft Java (Fabric, Java), the hub
- **Stand-in:** a client-side remote-player entity with interpolation, a name tag and chat bubbles.
- **Hits:** a mixin detects hits on the stand-in.
- **Anchor:** `/mp anchor` sets your zero point.
- **Install:**
  - The app adds a new profile to the official launcher, with its own game directory.
  - It installs Fabric through the Fabric installer's command-line mode.
  - The human still clicks Play: Microsoft sign-in always stays with the player.

### Second game, option 1: Terraria (tModLoader, C#)
- **Stand-in:** a dummy Player.
- **Mapping:** 1 tile ≈ 0.6 m, because a player is about 3 tiles tall.
- **Install:** copy and enable the `.tmod`. Steam Workshop comes later.

### Second game, option 2: Stardew Valley (SMAPI, C#)
- **Stand-in:** a custom character on the current map, in top view.
- **Install:** drop the mod into SMAPI's Mods folder. The app checks that SMAPI is installed.

All three run mods natively on macOS, so the MVP needs no compatibility layer.

## 5. Covalent's role (phase 2)
Covalent connects each person's own coding agent across machines. Here it does setup and builds adapters for new games, and Covalent itself needs no changes.

### Setup on both machines
Each owner grants their agent one workspace and one command:

```sh
harness access grant --agent <me>/lead --workspace ~/multiplayer-everything --command "multiplayer *"
```

The trusted `multiplayer` tool is what writes into game folders, with backups. The agent never edits game files directly.

1. Each agent runs `multiplayer doctor --json` and installs its own side.
2. The two agents agree on who hosts and pass the pairing code by Covalent message.
3. They run `multiplayer test --peer`: each adapter spawns test stand-ins at known positions and reports what it drew.
4. The agents fix misalignments (a flipped axis, wrong scale) without anyone driving a screen.

### New games
An **Adapter Kit** provides:
- templates for each mod loader;
- a checklist: find the player, spawn a stand-in, read the animation state;
- a conformance suite, which is the pass/fail check.

The agent on the machine that owns the new game builds the adapter, and the friend's agent is the test partner from the other machine. A passing adapter goes to the signed registry, and the next person gets one-click install.

## 6. Safety
- The app is notarized, and the adapter index is signed (ed25519), with checksums on every download. Fake "mashup" repos have been used to spread malware, so provenance matters.
- Mods listen only on the local machine, and only with the session token.
- Adapters install into separate profiles. Uninstall is verified by checksum.
- Offline or self-hosted only, with a blocklist of games that have anti-cheat.
- Game files are never distributed.

## 7. Acceptance

### MVP
1. On a fresh Mac: download → open with no Gatekeeper warning → install → pair → see the other player, all within 5 minutes.
2. The friend's movement looks smooth over home internet through the relay (target ≤250 ms perceived delay).
3. Hits, chat and emotes work in both directions.
4. Uninstall leaves both games byte-identical to before.
5. Killing the relay crashes neither game, and the session resumes when it's back.
6. Fuzzed or malformed messages crash neither game.

### Phase 2
1. Two agents, each with only the `multiplayer *` grant, complete setup on both machines.
2. They find and fix an injected axis flip using `test --peer`, with no human help.

## 8. Non-goals for v1
- Shared building (blocks and terrain edits)
- Items and trading
- More than two players
- Voice
- Consoles
- Competitive play
- Windows before phase 2
