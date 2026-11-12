# multiplayer-everything: build plan and estimates

Units are **engineer-days for one person working with coding agents**, given as ranges. Agents write most of the code. The bottleneck is humans launching games and judging how things look in game, and that time is included.

## Work packages

| # | Package | Days |
|---|---|---|
| W1 | Protocol schema, axes doc, fake test partner | 2–3 |
| W2 | Relay on Cloudflare Workers and Durable Objects: rooms, codes, limits, a "relay busy" state | 2–3 |
| W3 | Daemon and CLI: local socket, relay client, validation, `doctor` | 3–4 |
| W4 | Minecraft adapter (Fabric) | 5–7 |
| W5 | Second-game adapter: Terraria **5–7**, or Stardew Valley **4–6** | 4–7 |
| W6 | Installer: game detection, separate profiles, backup and uninstall | 4–5 |
| W7 | App UI, single-binary packaging, ad-hoc-signed DMG in CI, setup guide with screenshots | 3–4 |
| W8 | Two-Mac hardening and demo video | 3–4 |
| | **Phase 1, the downloadable MVP** | **26–37** |
| W9 | Covalent setup flow: agent INSTALL.md, `doctor --json`, `test --peer` | 3–4 |
| W10 | Adapter Kit and conformance suite | 4–6 |
| W11 | A third, 3D Windows game through CrossOver (BepInEx, UE4SS or SKSE); highest variance | 6–10 |
| W12 | Public release: registry signing, Windows app and its guide, Vercel download site, build attestations | 4–6 |
| | **Phase 2: Covalent, any game, public** | **17–26** |
| | **Total** | **43–63** |

**Later, not in the totals:** direct WebRTC connections, using Cloudflare STUN with a TURN fallback, so most sessions skip the relay (about 2–3 days).

The second-game choice moves the total by about a day. Stardew Valley is a little less work, because there's no combat feel to tune. Terraria makes the better demo.

## Milestones

- **M1 Hello** (W1, plus parts of W3 and W4): Minecraft talks to the fake partner on one Mac.
- **M2 First crossplay** (W2–W5): Minecraft ↔ the second game across two Macs through the relay, installed by hand. This is the first video.
- **M3 Downloadable** (W6–W8): DMG on GitHub Releases with the setup guide, pairing code, uninstall. Meets the MVP acceptance in SPEC.md.
- **M4 Agents set it up** (W9).
- **M5 Any game** (W10–W11).
- **M6 Public** (W12).

## Calendar time by capacity

These figures allow for coordination and testing on two machines.

| Scenario | M2 first video | M3 downloadable MVP | M6 public release |
|---|---|---|---|
| **Both full-time** (about 7 effective days/week; adapters built in parallel) | ~2.5–3.5 weeks | **~4–5 weeks** | ~6–9 weeks |
| **One person mostly** (5 days/week; the other tests on the second Mac) | ~4–5 weeks | **~6–8 weeks** | ~10–14 weeks |
| **Both part-time** (about 2 days/week each, about 3 effective days/week) | ~6–8 weeks | **~9–12 weeks** | ~15–21 weeks |

## How two people split it

- **Person A:** protocol (W1), Minecraft adapter (W4), installer (W6).
- **Person B:** relay (W2), daemon (W3), second-game adapter (W5).
- **Together:** app and packaging (W7), hardening and demo (W8).

Build it with Covalent, so the build is itself the showcase: each work package is a claim, and the protocol schema is a shared fact.

## Costs (check current prices)

- **Apple Developer Program:** not needed. The DMG is ad-hoc signed and not notarized, and the setup guide covers the one-time Open Anyway step.
- **Download site:** free on Vercel's Hobby plan; GitHub Releases hosts the files.
- **Relay:** $0 on Cloudflare's free plan, which covers about 28 hours of play a day across all rooms. Above that, the $5/month Workers Paid plan costs about $0.0005 per extra room-hour.
- **Windows code signing (phase 2):** optional. The default is the same as on Mac: unsigned, with a guide for SmartScreen's "More info → Run anyway".
- **Coding agents:** each person's own subscription.

## Risks

- **Minecraft updates often.** Pin one Minecraft and Fabric version pair, and build each supported version in CI.
- **Testing inside the game is slow and needs a human.** The fake partner (W1) and `test --peer` (W9) take most of it off people.
- **W11 can blow past its estimate.** A 3D Windows game under CrossOver is the riskiest package, but it's in phase 2 and the MVP doesn't need it.
- **Friction from the unsigned app:** some people will stop at macOS's block.
  - The guide has screenshots for each supported macOS version and is retested on every new macOS release, since Apple has changed this flow before (Control-click → Open stopped working).
  - Track how many people drop off at this step; buy notarization later only if it's high.
- **Copycat downloads:** an unsigned app with a "how to get past the block" guide is easy to imitate. Publish checksums and attestations, and name the only official download locations.
- **The relay's free daily limit:** once 100,000 requests are used, new ones fail until 00:00 UTC. The app shows "relay busy", and moving to the paid plan takes minutes if a launch takes off.
- **Model refusals:** low risk. Every MVP adapter uses an official mod API, not reverse engineering.

## Open decisions

- The second game: Terraria or Stardew Valley.
- Who takes Person A and who takes Person B, and the capacity scenario.
- The license, decided before open-sourcing.
