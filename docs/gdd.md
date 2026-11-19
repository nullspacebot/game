# NULL SPACE: game design (MVP)

Nov 19, 2026 · @Gary

## Overview

NULL SPACE is a turn-based trading game played entirely by posting commits to a bot account on dev-null.sh. This document calls it @nullspace, but the handle isn't settled yet (see [Bot account](technical.md#bot-account) in the technical design). The MVP proves one loop: move around a hex map, harvest and trade three goods for profit, and meet other players near the start.

The setting: everything ever piped to /dev/null ends up in an endless void of drifting junk. Each player captains a small salvage ship.

Design principles:

- **Play by mail, not real time.** Orders are collected and resolved together on a fixed tick, every 6 hours. Each player gets at most one reply per tick.
- **One thread per captain.** All of a player's orders are replies in their own thread, so the public timeline only sees join posts and one daily bulletin.
- **Fair and repeatable.** Turn order is shuffled every tick, and the same orders always give the same result.

How it's built is in the [technical design](technical.md).

## MVP scope

The MVP is trading only, with no combat and no building. Everything not listed as in is out.

| Area | In the MVP | Out of the MVP |
| --- | --- | --- |
| Joining | `@nullspace join`, instant welcome reply | Invites, referral rewards |
| Map | Seeded endless hex map, Hub at the centre, personal fog of war | Hex naming, player-written lore |
| Actions | `move`, `scan`, `harvest`, `buy`, `sell`, `offer`, `accept`, `name`, `status`, `help` | Building, combat, crafting |
| Economy | 3 goods, stations with supply-driven prices | Production chains, contracts board |
| Ship | Fuel per tick, 3 upgrades bought at the Hub | Multiple ships, crew |
| Other players | Seeing ships in scans, trades offered in the same hex | Chat, guilds, piracy |
| Risk | One table of deep-void events | Fightable pirates |
| Output | Join reply, turn reports with a map image, daily bulletin | Web map viewer |
| Seasons | One open-ended season, reset by hand | Scheduled seasons |

## How a player plays

A player joins with one top-level commit, then plays by replying in that commit's thread. That thread is their captain's log.

1. **Join.** The player posts a top-level commit containing `@nullspace join`. Any other text is allowed.
2. **Welcome.** The bot replies to that commit within a few minutes, outside the tick. The reply gives the ship's name, starting position, hold, fuel, credits, Hub prices, a map image and the command list.
3. **Orders.** Before the next tick, the player replies anywhere in their own log thread with a post containing a fenced code block of orders. Text outside the block is ignored by the game.
4. **Tick.** Four times a day, 6 hours apart, the engine resolves every player's orders together. The exact times are to be confirmed to suit players' timezones. The working default is 00:00, 06:00, 12:00 and 18:00 UTC.
5. **Report.** The bot replies to the player's latest orders post with a turn report and a map image.
6. **Repeat** from step 3.

Rules:

- Every bot post gives the next tick as a countdown ("in 3h 57m") as well as a UTC time, so players don't have to convert timezones.
- Ticks run every 6 hours. A player who sends no orders for a tick gets no reply, and their ship stays where it is.
- Commits can't be edited on dev-null.sh, so a player corrects orders by posting again. The latest orders post before the tick wins. The report says how many earlier posts were ignored.
- An orders post that is still pending moderation at tick time isn't visible to the bot, so it counts for the next tick.
- A deleted orders post is ignored.
- Posts that mention @nullspace outside a registered log thread get no reply, except a new `join`.
- A player who joins twice keeps their first log thread. The bot replies once to the second join, pointing to the first thread.

## Orders

The engine reads only the first fenced code block in a post, one order per line, at most 10 lines. Orders run top to bottom.

| Order | Example | Effect | Fuel |
| --- | --- | --- | --- |
| `move <dir> [<dir> ...]` | `move ne ne ne` | Moves one hex per direction: `n`, `ne`, `se`, `s`, `sw`, `nw` | 1 per hex |
| `scan` | `scan` | Charts every hex within scanner radius and lists what's there | 0 |
| `harvest` | `harvest` | Takes goods from the field in the current hex. Once per tick | 0 |
| `buy <good> <qty>` | `buy cycles 40` | Buys from the station in the current hex | 0 |
| `sell <good> <qty\|all>` | `sell cycles all` | Sells to the station in the current hex | 0 |
| `offer @user <qty> <good> for <qty> <good\|cr>` | `offer @alice 18 entropy for 400 cr` | Opens a trade offer (see Player trades) | 0 |
| `accept @user` | `accept @you` | Accepts that player's open offer | 0 |
| `upgrade <part>` | `upgrade hold` | Buys the next upgrade level. Hub only | 0 |
| `name <ship name>` | `name The Segfault` | Renames the ship, up to 32 characters | 0 |
| `status` | `status` | Adds a full ship and market summary to the report | 0 |
| `help` | `help` | Adds the command list to the report | 0 |

Parsing rules:

- Commands and goods are case-insensitive. Extra spaces are ignored.
- A line that fails, for example an unknown command, not enough fuel, credits or hold space, or no station here, is skipped. The report says why. Every other line still runs.
- An unknown command gets a "did you mean" hint when it is within edit distance 2 of a real one.
- A `move` that runs out of fuel stops at the last hex it could reach.
- `buy` with more than the hold or credits allow buys as many as fit, and the report says so.
- `upgrade` covers the three parts in the Ship section. A post with no code block is not an orders post and gets no reply.

## The world

The map is an endless flat-top hex grid, generated from a season seed as players explore. There is no edge: the further out a ship goes, the emptier and richer the void gets.

- **Directions:** six, matching the flat-top hexes: `n`, `ne`, `se`, `s`, `sw`, `nw`.
- **Ring:** a hex's distance from the Hub at (0,0). The ring sets how rich and how crowded it is.
- **Hex types:** each hex is empty, a station or a field. The same seed always produces the same map.
- **Density falls off with distance:** stations are common near the Hub and rare far out, while fields get richer and rarer goods appear.
- **Spawn:** new ships start on a random empty hex in ring 1 or 2, so early players meet.
- **Fog of war is per player.** Each player sees only hexes they have charted, by scanning or passing through.

Starting generation table, to be tuned in playtests:

| Ring | Station chance | Field chance | Field goods |
| --- | --- | --- | --- |
| 0 | The Hub | — | — |
| 1–4 | 12% | 20% | Mostly cycles |
| 5–9 | 6% | 18% | Cycles and entropy |
| 10–15 | 3% | 15% | Entropy and legacy |
| 16+ | 1% | 12% | Mostly legacy, richest fields |

Station and field names come from seeded word lists with a developer theme, for example Grepstone Relay or Kernel Panic Depot.

## Economy

Profit comes from buying where a good is cheap and selling where it is scarce. Every trade moves the price, so players compete through the markets.

| Good | Base price | Found in | Units per harvest |
| --- | --- | --- | --- |
| Cycles | 11 cr | Near the Hub | 20–30 |
| Entropy | 28 cr | Middle rings | 12–20 |
| Legacy | 45 cr | Far rings | 6–12 |

**Stations**

- The Hub buys and sells all three goods, with a spread of about 15% between its buy and sell prices.
- Every other station is seeded with a role per good: it **sells** the good (cheap, below base), **buys** it (pays above base), or doesn't trade it. Each station trades two of the three goods.
- A station's price for a good starts from base × a seeded factor: 0.6–0.85 where it sells, 1.25–1.7 where it buys.

**Prices respond to trades**

- Each unit a player sells to a station lowers that station's price for the good by 0.5%. Each unit bought raises it by 0.5%.
- Prices are clamped to 50%–200% of base.
- Each tick, every price moves 25% of the way back to its starting value.
- Trades apply one unit at a time, so a large sale gets a slightly worse average price.

**Fields**

- A field holds a seeded stock, for example 400–900 units, that only goes down in the MVP.
- `harvest` takes a random amount in the good's range, capped by hold space and the field's stock.
- An empty field stays empty for the rest of the season. Scans show a field's stock as rough bands: rich, worked, thin, empty.

**Starting ship:** 500 cr, empty hold.

## Ship

Every ship starts with a 40-unit hold, 12 fuel and a scanner radius of 2. Fuel refills to full at the start of each tick and can't be stored or bought.

| Upgrade | Level 1 (start) | Level 2 | Level 3 |
| --- | --- | --- | --- |
| `hold`: cargo capacity | 40 units | 60 units, 900 cr | 80 units, 2,000 cr |
| `engine`: fuel per tick | 12 | 16, 1,200 cr | 20, 2,800 cr |
| `scanner`: scan radius | 2 | 3, 650 cr | 4, 1,600 cr |

- Upgrades are bought with `upgrade <part>` at the Hub only and take effect immediately.
- Ships get a random two- or three-word name at join, for example The Segfault. `name` changes it.
- **Net worth** = credits + cargo valued at the Hub's buy prices + half of what was spent on upgrades. It ranks the leaderboard.

## Player trades

Two players who meet in the same hex can swap goods or credits. An offer stays open for two ticks, so the other player doesn't have to be online in the same 6-hour window.

- **Offer:** `offer @alice 18 entropy for 400 cr`. Both ships must be in the same hex when the offer is made. Each side can be a good or `cr`.
- **Notification:** the @mention in the offering post notifies Alice through the platform. Her next turn report also lists the open offer.
- **Accept:** `accept @you`, from Alice's own log thread. It works in the same tick as the offer, or either of the next two.
- **Settlement:** the ships don't need to share a hex any more, because either may have moved on since the offer. The lore glosses over how the goods cross the gap. Each ship must hold what they're giving and have room for what they get. The swap is all or nothing.
- **Limits:** one open offer from each player to each other player. A new offer replaces the old one. An offer expires after two ticks.
- Both reports say whether the trade settled, failed (and why) or expired.

## Deep-void events

Each hex a ship enters in ring 5 or beyond has a 10% chance of triggering one event. At most one event per ship per tick. Events add risk to long hauls.

| Roll (d100) | Event | Effect |
| --- | --- | --- |
| 1–30 | Drifting cache | +5–15 of a random good, capped by hold space |
| 31–50 | Solar wind | The rest of this tick's moves cost double fuel |
| 51–70 | Hull breach | Lose 10–25% of one good in the hold |
| 71–85 | Scrapper pirates | Lose 15% of credits, or 20% of cargo if they're worth more |
| 86–100 | Old beacon | Charts every hex within radius 4 for free |

Each event has two or three flavour-text variants so the reports don't repeat. Pirates can't be fought in the MVP.

## Bot output

The bot writes three kinds of post, all from hand-written templates. Every post must fit within dev-null.sh's limit of 4,000 characters and three images.

| Post | When | Where | Contents |
| --- | --- | --- | --- |
| Welcome | Within minutes of a `join` | Reply to the join commit | Ship name, position, hold, fuel, credits, Hub prices, map image, command list, time of next tick |
| Turn report | After each tick, only for players who sent orders | Reply to the player's latest orders post | One line per order (✓ done or ✗ skipped, with the reason), scan results, events, trade results, open offers to the player, status line, map image, time of next tick |
| Daily bulletin | After the first tick of each UTC day | Top-level commit | Player count, hexes charted, biggest price moves, nearly empty fields, top 5 by net worth, trades settled |

**Turn report format** follows the thread mock-up: a monospaced block with a `TICK n` header, `✓` and `✗` lines, then `SCAN`, `NEARBY` and `EVENTS` sections, then one status line: `HOLD 18/40 entropy · FUEL 11/12 · 780 cr`.

**Map image**

- A PNG of the player's charted hexes, centred on their ship and roughly 9 hexes across.
- Distinct fills for the Hub, stations, fields, charted empty hexes and uncharted void. Other ships are dots, and this tick's route is a dashed line.
- Short labels inside hexes (HUB, station initials, the field's good).
- The image description is generated alt text listing what is in view, for example "Grepstone Relay here, entropy field 1 hex north, Alice's ship 2 hexes south".

**Voice:** short, dry, slightly ominous. Each template has a few variants chosen by the seeded RNG so reports don't read identically.

## Open questions

- [ ] Fix the tick times once we know where most players are. The working default is 00:00, 06:00, 12:00 and 18:00 UTC.
- [ ] Decide whether players must share a hex to make a trade offer, or can trade with anyone they've met.
- [ ] Tune every number here (prices, harvest sizes, upgrade costs, event odds) in the balance pass described in the [technical design](technical.md#build-plan).

## Further iterations

Possible improvements after the MVP, not yet specified:

- Reasons to write flavour text: log text broadcast as chatter in nearby players' scans, naming and describing discovered hexes, stations that remember regulars, and a hand-picked Log of the Day in the bulletin
- Outposts and base building on claimed hexes
- Fightable pirates (PvE)
- Player combat and interdiction, with commit-reveal orders so the last player to post can't read and counter the others
- Guilds built on dev-null.sh Orgs
- Scheduled seasons with map resets and season rewards
- More goods and production chains
- A public contracts board for buy and sell orders
- Field regrowth and shifting markets
- A read-only web map viewer
