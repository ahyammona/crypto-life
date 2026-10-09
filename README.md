# Crypto Life

A browser 3D life-sim for a crypto community, modelled on the *Lagos Life* game (reference screenshots in `docs/reference/`). You live in a home, keep six needs topped up, travel a city map by Trek / Keke / Okada / Cab / Lambo, work, socialise, gamble in Rug Alley, and trade paper-money tokens from your phone. The wallet's **Launchpad** tab streams live (replayed) pump.fun-style token launches from the Rust market feed.

The Rust workspace is the backbone: game rules and world data, bonding-curve math, wire protocol, replay harness and the server. The client is one plain-JS file using three.js. **Paper money only; nothing touches a chain or a wallet.**

## Run it

```sh
# Rust 1.85+ (edition 2024). Built and tested here on 1.91.
cargo run -p server -- demo.pwrec --addr 127.0.0.1:8099
# open http://127.0.0.1:8099        (add ?reset to wipe the saved game)
```

`demo.pwrec` is a deterministic synthetic market (regenerate: `cargo run -p replay -- gen --seed 7 --tokens 200 --out demo.pwrec`).

```sh
cargo test --workspace                                   # all crates
cargo run -p life-core --example dump_config > docs/config.example.json
```

## Layout

| Path | What it is |
|---|---|
| `crates/life-core` | **Game rules + world data**, pure Rust, 8 tests. `Needs` (decay/clamp/mood), `Action`, `Player::apply` (cost, effects, gamble with an injected roll), and `GameConfig::default_world()`: places, transports, shop, tokens, home furniture actions, decay rates. |
| `crates/server` | `cl-server` (axum). Serves the client, `/api/config`, and streams the market recording over `/ws`. |
| `crates/server/web/life.html` | **The game client** (single file, ES module). |
| `crates/curve`, `crates/protocol`, `crates/replay` | Pump.fun-style bonding-curve math, versioned binary wire types, `.pwrec` recorder/player/checker. Unchanged foundation: see `docs/PUMPWORLD_FOUNDATION.md`. |
| `crates/server/web/pumpworld.html`, `index.html` | Earlier prototypes (walkable Pump.fun/PumpSwap street at `/pumpworld`, top-down view at `/map`). `world.html` is unused. |

## How the pieces talk

```
life-core::GameConfig ──GET /api/config (JSON)──▶ life.html builds world from it
demo.pwrec ──cl-server /ws (JSON frames: created | trade | graduated | risk | reset)──▶ Launchpad tab
```

* **Single source of truth for content.** To add a place, transport, shop item, token or furniture action, edit `GameConfig::default_world()` in `crates/life-core/src/lib.rs`. The client builds map pins, the travel sheet, the Buy tab, action buttons and the wallet token list from that JSON. A shop item only needs a matching home object if you want one visible (see below).
* **JSON names are a contract.** Append fields, don't rename or remove (same rule as the wire protocol). `CONFIG_VERSION` is 1.
* **Live feed.** `life.html` opens `/ws?speed=40`. `created` adds a launchpad row, `trade` updates market cap and progress, `graduated` marks it, `risk` sets a warning, `reset` (loop restart) is ignored on purpose so holdings stay valid. Price shown in the Launchpad is market cap in SOL; a paper buy of $100 buys `100 / mcap` units, so value tracks market-cap movement. It is a game proxy, not a real price.

## Client structure (`life.html`)

One `<script type="module">`. three.js is the bundled r186 ES module served at `/three.min.js` (`import * as THREE from "/three.min.js"`). Notes:

* `THREE.ColorManagement.enabled=false` and light intensities are multiplied by `Math.PI` to keep r128-era look under r186's physical lights. Don't add `PointLight`s without handling units.
* Views: `home` (isometric cutaway room, tap furniture to walk there and use it) and `map` (tilted city, drag to pan, DOM pins projected each frame). `setView()` toggles the two groups.
* Home furniture is a `Group` with `userData.act` = a key in `CFG.home_actions` (`sleep eat relax wash wc phone`). Shop items with ids `rig chair wall lambo` toggle pre-built hidden furniture in the `extra` map; **a new shop id with no `extra` entry still works but shows nothing at home.**
* Generic action runner `runAction(a)` handles cost, needs delta, cash, minutes, gamble, fade text, and `opens_phone`. It mirrors `life_core::Player::apply`.
* Sheets: `#sheet` (travel, place, buy) and `#phone` (wallet). `showEl()` opens one and adds the ✕ button and the scrim; `closeAll()` closes both. The bottom nav sits above sheets (z-index 12) and the Phone button toggles the wallet. **Fixed this session:** the wallet used to cover the nav with no way out.
* Wallet rows render once per tab/list change; prices update in place every tick (`updPhone`) so buttons stay clickable. Launchpad re-renders at most every 1.5s when the visible token list changes.
* Save: `localStorage['crypto-life-v1']` every 4s, loaded on boot (`?reset` skips). Holdings, cash, needs, owned items, location, quest step.
* Time: 1 real second = 2 game minutes; need decay comes from `CFG.decay` per game minute. Passive income is `S.rig` dollars per game hour.

## Status

Working and tested headlessly (Playwright/Chromium, no console errors): home scene, map, travel sheet, place actions, Buy, wallet with Wallet and Launchpad tabs, live feed, wallet close paths, save/load. Rust: `cargo test --workspace` passes in full (life-core has 8 tests; the server builds and serves `/`, `/api/config`, `/ws`).

## Known gaps (honest list)

1. **Game state lives in the browser.** `life_core::Player` exists and is tested but the server doesn't run it. The JS and Rust rules are duplicated (mood thresholds, `runAction` vs `Player::apply`). Next step: compile `life-core` to WASM (it is `forbid(unsafe_code)`, serde only) or move state server-side.
2. **Not multiplayer.** "👀 17.2m" and "154k online" are cosmetic (online wobbles with the clock). No neighbours, chat or shared rooms yet. The roadmap items `hub`, presence and rooms in `docs/PUMPWORLD_FOUNDATION.md` are the intended route.
3. **Prices are paper.** Wallet tokens are a local random walk (`tick()` in the client); Launchpad prices come from the synthetic recording. No real chain data, wallet connection or transaction signing. The foundation doc's "transaction firewall" plan applies before any of that.
4. **Home is fixed.** Layout is hard-coded; only the four shop items change it. No furniture placement, no other homes.
5. **Placeholder content.** Quests (5 steps, text in `ui()`), place actions, token symbols and prices are my invention. The map is blocks and trees, not the detailed streets of the reference.
6. **Not run here:** `cargo fmt` and `cargo clippy` (not installed in the build sandbox). Run both before committing; CI config from the original repo isn't in this zip. No mobile touch testing beyond a responsive layout.
7. The first render of the bundled three.js build assumes WebGL2. No fallback.

## Good next steps

1. `life-core` to WASM, delete duplicated JS rules, add save-game serialisation in Rust.
2. Quests and goals as data in `GameConfig`, with rewards.
3. Real avatar and animated NPCs on the map; interiors for the café, bar and Pump.fun floor.
4. Hub/presence so other players appear on the map (build on `/ws` + `protocol`).
5. `venue-pumpfun` and `signals` from the foundation roadmap for real launches and risk scoring.

## Reference

`docs/reference/lagos-life-1..5.png`: the street view, two map views, the travel sheet ("Go · ₦500") and the home cutaway, in the order the user sent them. The UI here copies that layout: top status pill, needs panel bottom-left, quest chip top-left, four-tab nav, map chips, bottom travel sheet.
# crypto-life
