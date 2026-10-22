# Dolores' Boosting Contracts (kb_boosting): setup guide

Vehicle boosting contracts by Kerbside. Players pick up contracts from a board, find the car, kill the tracker,
and drop it off for cash and reputation. Reputation unlocks better tiers, D up to S.

Version 1.0.3. Works with ESX Legacy, QBCore, Qbox, or no framework at all.

## What it does

- A voiced fixer. Dolores, a foul-mouthed Southern car dealer, phones the player with the offer, warns them when the
  cops are on it, and lets them know how they did. 740 recorded lines across twenty-four moods, on a call screen with
  subtitles. Her calls ring first with the game's own phone ring, her voice has a light telephone filter, and the
  car radio goes quiet while she talks. Turn the voice off, change the volume, keep the radio on, switch
  off the ring or the filter, use your own ring sound, or rename her in `Config.Fixer`.
  She works through every line of a kind before she repeats one, and remembers where she got to between sessions.
- Crews. Players team up from the board or with `/crew invite <id>` (up to `Config.Crew.maxSize`). Any job one of
  them takes brings the free crew mates along: everyone sees the car, the trackers and the drop-off, the pay is split
  evenly, and everyone gets the reputation and the cooldown. If someone leaves mid-job the rest carry on.
- VIN scratch. On the tiers you pick (`Config.VinScratch.tiers`, B and up by default) the player can pay to keep the
  car at the drop-off instead of handing it over: it gets a fresh plate, goes in their garage (with keys on
  qbx_vehiclekeys or qb-vehiclekeys), and is remembered as scratched. Police can check a car with `/vincheck`. Other resources can ask with
  `exports.kb_boosting:IsVinScratched(plate)`.
- Five tiers (D, C, B, A, S) with their own cars, pay, time limit, tracker count, buy-in and police requirement.
- Contracts turn up on a timer and when a player opens an empty board. Each player has a small queue; contracts expire.
- The server picks the car, plate, spawn point and drop-off. The client only asks for the next step.
- Tracker hacks: type a key sequence against the clock. The server checks the answer and the timing, and turns down
  answers that come back faster than a person can type.
- Search phase: the map shows a circle that shrinks in steps as the player gets closer. The exact car blip only
  appears when the player is within `Config.RevealDistance` of the car.
- Police with a listed job get tracker pings on the map while a tracker is live, with part of the plate. If
  ps-dispatch or cd_dispatch is running, the alert goes through it instead (`Config.Dispatch`).
- Optional buy-in refund when a job fails (`Config.RefundOnFail`, off by default).
- Delivery is checked on the server: the car has to be inside the zone, the player next to it, and the run has to take
  a believable amount of time for the distance.
- Payout scales with the car's condition. Wrecking it fails the contract.
- Reputation, contracts done and failed, and money earned are saved per character.
- Cooldowns after a job, after a failure, after abandoning, and optional server-wide cooldowns on the top tiers.

## Requirements

- A FiveM server with OneSync on (the default on current builds). The cars are created by the server.
- Optional: `ox_lib` (notifications), `ox_target` or `qb-target` (laptop spots), `ox_inventory`, `oxmysql`.
  None of them are required.

## Install

1. Put the `kb_boosting` folder in your server's `resources` folder.
2. Add `ensure kb_boosting` to `server.cfg`, **after** your framework (`es_extended`, `qb-core` or `qbx_core`)
   and after `oxmysql` / `ox_lib` / `ox_target` if you use them.
3. Give staff the admin commands:
   ```
   add_ace group.admin command.boost:reset allow
   add_ace group.admin command.boost:info allow
   add_ace group.admin command.boost:give allow
   ```
4. Start the server and type `boost:selftest` in the server console. Every line should end in `ok`.
   The test uses a throwaway profile; remove it after with `boost:reset selftest:console`.
5. In game, type `/boosting` to open the board. Out of the box it opens anywhere. To make players go to a laptop,
   set `Config.Laptop.zoneCheck = true` (and `Config.Target = 'auto'` for a target prompt there).

## Configuration

Everything lives in `config.lua`. The main switches:

| Setting | Values | What it does |
| --- | --- | --- |
| `Config.Framework` | `auto`, `esx`, `qb`, `qbox`, `standalone` | `auto` checks for qbx_core, then qb-core, then es_extended |
| `Config.Notify` | `auto`, `ox_lib`, `esx`, `qb`, `native` | How messages are shown |
| `Config.Target` | `none`, `auto`, `ox_target`, `qb-target` | Adds laptop spots at `Config.Laptop.zones` |
| `Config.Inventory` | `auto`, `ox_inventory`, `framework`, `none` | Used for the optional tablet item and tier items |
| `Config.Persistence` | `json`, `oxmysql` | JSON writes `data/progress.json`; oxmysql makes tables `kb_boosting_progress` and `kb_boosting_vins`. Switching does not copy existing progress across |
| `Config.MoneyAccount` | per framework | Which account pays out and takes buy-ins |

Tiers are in `Config.Tiers`. Each one sets `minRep`, `weight` (how often it is rolled), `reward = { min, max }`,
`buyIn`, `repGain`, `repLoss`, `timeLimit` (seconds), `hacks` (trackers to kill), `hackLength`, `hackTime` (ms),
`minPolice`, `policeAlert`, `globalCooldown` and `vehicles` (spawn names). Add `requiredItem = 'name'` to a tier
to demand an item before it can be taken.

Spawn points and drop-offs are in `Config.SpawnPoints` and `Config.DropOffs`. Walk every point in game before
going live and add your own; a busy server wants at least ten of each.

To gate the board behind an item, set `Config.Laptop.requiredItem = 'your_item'`. The board then only opens for
players holding it.

By default the board opens anywhere. With `Config.Laptop.zoneCheck = true` it opens only at a laptop spot: the
server checks the player's position against `Config.Laptop.zones` plus `Config.Laptop.zoneDistance` metres, and
turns the request down otherwise.

### Tablet item

`Config.Laptop.tabletItem` (default `boosting_tablet`) is a usable item that opens the board anywhere. The script
registers it as usable, but your inventory has to know the item exists. Add it once, then restart the server:

- ESX (default inventory): ESX ignores items missing from its `items` table, so run
  `INSERT INTO items (name, label, weight) VALUES ('boosting_tablet', 'Boosting Tablet', 1);`
- QBCore: add to `qb-core/shared/items.lua`:
  `boosting_tablet = { name = 'boosting_tablet', label = 'Boosting Tablet', weight = 500, type = 'item', image = 'tablet.png', unique = false, useable = true, shouldClose = true, description = 'Work orders' },`
- ox_inventory (ESX, QBCore or Qbox): add to `ox_inventory/data/items.lua`:
  `['boosting_tablet'] = { label = 'Boosting Tablet', weight = 500, stack = false, close = true },`
  On Qbox the script registers it with qbx_core. If using it does nothing, add
  `server = { export = 'kb_boosting.boosting_tablet' }` to that item.
- Standalone: there is no inventory, so the command in `Config.Laptop.tabletCommand` (default `/boostingtablet`)
  stands in for using it.

Set `Config.Laptop.tabletItem = false` to turn it off.

### Refunds, dispatch and the search circle

- `Config.RefundOnFail`: when true, a failed job pays the buy-in back if the reason is true in
  `Config.RefundReasons` (timeout, wrecked car and failed hacks by default; abandoning is off by default, and flagged runs never pay back).
- `Config.RefundOnRestart`: restarting kb_boosting or the server mid job removes the car and ends the job. When true
  (default), the buy-in goes back to whoever paid it and everyone on the job is told why it stopped.
- `Config.Fail.onDisconnect`: what a crash or disconnect does to an active job. `'cancel'` (default) ends it with no
  reputation hit, the car is removed and the cancel cooldown applies when the player is back; `'fail'` treats it as a
  failed job (reputation loss, fail cooldown). Either way the buy-in stays with the fixer.
- `Config.Dispatch`: `auto` uses ps-dispatch or cd_dispatch when started and the built-in police blips otherwise;
  `none` always uses the built-in blips. Using a different dispatch script? Email us and we'll add it.
  ps-dispatch filters alerts by job *type* (`leo` by default), so on QBCore/Qbox add `'leo'` to `Config.Police.jobs` if your police job is typed that way.
- `Config.SearchSteps` and `Config.RevealDistance` set how fast the search circle shrinks and how close a player
  has to be before the car shows on the map.

The generation timer can be changed without editing the file: `set kb_boosting_tick 120` in `server.cfg`.

### Crews and VIN scratch

- `Config.Crew`: `enabled`, `maxSize`, the `command` name, how long an invite lasts (`inviteSeconds`), and two
  distances that are off (0) by default: `inviteDistance` (how close you must be to invite) and `shareDistance`
  (crew mates further than this from the car at hand-over get no share).
- `Config.VinScratch`: `enabled`, which `tiers` allow it, `cost` (up to 1 is a share of the payout, above 1 a flat
  price), `repShare` (how much of the tier's reputation a kept car earns), the `garage` each framework stores it in,
  `keys`, and `give`, a function you can set to hand the car to your own garage script instead. Standalone servers
  have no garage, so the car simply stays with the player. `policeCheck` and `checkCommand` control `/vincheck`.

### Standalone servers

With no framework, players get a credit balance kept with their progress (shown on the board). To pay into your own
economy as well, set `Config.Standalone.payoutEvent` to a server event name; it is called as
`TriggerEvent(name, source, amount, reason)`. Police are players with the ace `kerbside.police`
(`add_ace group.police kerbside.police allow`). If you have no police, set `minPolice = 0` on every tier.

### Anti-cheat hooks

Rejected requests are printed to the console (`Config.AntiCheat.logRejects`). Set `Config.AntiCheat.onReject` to
a function to send them to your own logging or ban tool.

`config.lua` is a shared file, so every player's game downloads it. Never put a webhook URL, API key or
password inside `onReject`, `Config.VinScratch.give` or any other config function. Have the function trigger a
server event or call an export from your own server-only resource, and keep the secret there.

## Commands

| Command | Who | What |
| --- | --- | --- |
| `/boosting` | players | Open the board (anywhere, or at a laptop spot with `zoneCheck`; rename or turn off in `Config.Laptop.command`) |
| `/boostingtablet` | players | Standalone only: stands in for using the tablet item |
| hack key (default G) | players | Work on the tracker while in the car; players can rebind it in Settings > Key Bindings |
| `/crew invite\|accept\|decline\|leave\|kick [id]` | players | Manage a crew (also on the board) |
| G at the drop-off | players | Pay to scratch the VIN and keep the car, on tiers that allow it |
| `/vincheck` | police | Check the nearest car for a scratched VIN |
| `boost:reset <server id or identifier>` | ace `command.boost:reset` | Wipe a player's boosting progress |
| `boost:info <server id or identifier>` | ace `command.boost:info` | Show a player's progress |
| `boost:give <server id or identifier> [D/C/B/A/S]` | ace `command.boost:give` | Put a contract on an online player's board now (events, testing) |
| `boost:selftest` | server console | Run the server checks |

## Events for other resources

Server to client, so you can hook HUDs or phones: `kb_boosting:client:accepted`, `kb_boosting:client:deliver`,
`kb_boosting:client:ended (id, outcome, reason, payout)`, `kb_boosting:client:policePing`.

## Files you can edit

`config.lua`, `locales/`, and `bridge/` stay readable under escrow. If your framework is a fork, the bridge files
are where to adapt it: each one exposes `GetPlayer`, `GetIdentifier`, `AddMoney`, `RemoveMoney`, `HasItem`,
`Notify`, `GetJob` and `RegisterUsable`.

Keep the `data/` folder when you update, or saved progress is lost.

## Support

Email kerbsidestudios@gmail.com and we'll reply within 24 hours.

- Bugs in the resource are fixed for free, and fixes ship as updates.
- Install help is included: if the resource will not start or the board will not open, ask.
- Files you have edited are not supported. If you changed code, send a clean copy of the problem or put the
  original file back before you ask. Settings in `config.lua`, `locales/` and `bridge/` that this guide says you may
  change are fine to ask about; your own edits to the logic are not.

## Legal

kb_boosting is made by Kerbside. It is not approved, sponsored, or endorsed by Rockstar Games or Take-Two
Interactive. It is not an official Rockstar product.

The fonts in `html/fonts` (Barlow, Barlow Condensed, Space Mono) are used under the SIL Open Font License 1.1; the
licence texts are next to them.
