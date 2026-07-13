---
name: ha-context-map
description: >-
  Use when an agent keeps guessing wrong entity ids, when starting recurring
  agent work against a Home Assistant instance, or when asked to "map my Home
  Assistant", "build an entity inventory", "document my smart home", or "why
  does the agent keep using entities that don't exist". Covers writing and
  maintaining the instance map file: entity inventory, add-ons, integrations,
  helpers, notify targets, and a running Gotchas list, plus the provenance
  checks that keep it honest. Not for making the initial connection
  (ha-connect) or writing automations (ha-automations).
compatibility: Any Home Assistant install reachable per ha-connect.
---

# HA Context Map

The part people skip. An agent working against a Home Assistant instance
without a map will guess entity ids, and Home Assistant fails silently
around bad ids often enough that guessed automations *look* fine and never
fire. One markdown file fixes this. Assumes the connection lanes from
`ha-connect`.

## Why a map file beats querying every time

`/api/states` on a mature instance returns hundreds of entities with
machine-generated names (`light.living_room_2`, because the first
`living_room` was deleted and re-added). The map is where a human confirms,
once, that `light.living_room_2` **is** the actual living room lamp, and
where the agent learns it without burning a discovery round-trip every
session. State is queryable; *meaning* is not.

Keep it wherever your agent reads context from: the repo where you do HA
work, or directly on the box as `/config/CLAUDE.md` (or `AGENTS.md`) so any
agent that lands there finds it.

## What goes in it

A section per category, **every section dated**. Suggested skeleton:

```markdown
# Home Assistant instance map (snapshot YYYY-MM-DD)

## Host
- Host: <HA_HOST>, Web UI :8123, SSH via Terminal & SSH add-on
- Core / OS version (from `ha core info`)
- Token: env var HA_TOKEN in <path>; NOT stored in this file

## Add-ons
- Terminal & SSH, Mosquitto, ... (from `ha addons`)

## Integrations that matter
- e.g. Hue, MQTT, a camera platform, a utility-price integration

## Key entities (verified by hand: the point of this file)
| Entity | What it actually is |
|--------|---------------------|
| light.living_room | Floor lamp, living room |
| media_player.tv | The TV automations may power off |
| climate.thermostat | Single thermostat; in cool mode target = `temperature` attr |

## Helpers
- input_boolean.away_mode, input_number.price_threshold, ...

## Notify targets
- notify.mobile_app_<phone>: the CURRENT one; old phones leave dead services

## Automations index
- count + how to list them (don't inline all of them; they rot fastest)

## Gotchas (append-only, dated)
- YYYY-MM-DD: <thing that burned you>
```

### Building the inventory without clicking through the UI

```bash
source ~/.config/ha/env

# Every entity id, grouped by domain
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/states" \
  | jq -r '.[].entity_id' | sort | awk -F. '{print $1}' | uniq -c | sort -rn

# All entities in one domain, with friendly names (meaning candidates)
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/states" \
  | jq -r '.[] | select(.entity_id | startswith("media_player."))
           | "\(.entity_id)\t\(.attributes.friendly_name // "-")"'

# Helpers
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/states" \
  | jq -r '.[].entity_id' | grep -E '^(input_|timer\.)'

# Add-ons and versions
ssh root@<HA_HOST> 'ha addons'

# Automations: count + aliases (UI-managed automations live in automations.yaml)
ssh root@<HA_HOST> 'grep -cE "^- id:" /config/automations.yaml'
ssh root@<HA_HOST> 'grep -E "^- id:|^  alias:" /config/automations.yaml'
```

The queries produce the *candidates*; the human pass that says which
`media_player` is the actual family TV is what makes the file worth having.
Do that pass once, in the file, and mark it verified.

## The Gotchas list

The highest-value section, and the one nobody writes. Every instance
accumulates local traps that no doc can predict. Real examples of the *shape*
of entry that belongs there:

- "`light.kitchen` is a group of only two of the four kitchen lights."
- "The lamp on the switched outlet reads `unavailable` when the switch is
  off. Normal. Do not flag it."
- "The old phone's `notify.mobile_app_*` service no longer exists;
  automations calling it abort silently at that step."
- "Renamed entities keep their old id unless you rename the *entity id*, not
  just the friendly name."
- "`unavailable` on an `automation.*` entity means an orphaned registry
  entry, not a broken automation."

Append with a date, never rewrite history. When an agent hits something that
cost more than five minutes, the fix isn't finished until the gotcha is in
the map.

## Keeping it honest: provenance checks

A map that silently rots is worse than no map: the agent trusts it. Two
rules:

1. **Date every snapshot section.** "Add-ons (as of 2026-07-01)" tells a
   future agent exactly how much to trust it.
2. **Ship re-verify one-liners with the claims.** For each volatile section,
   include the command that checks it, so any session can cheaply re-run:

```markdown
## Provenance
- Versions: `ssh root@<HA_HOST> 'ha core info'`; expect 2026.x per Host section
- API alive: `curl -s -H "Authorization: Bearer $HA_TOKEN" http://$HA_HOST:8123/api/`
- Automation count: `grep -cE "^- id:" /config/automations.yaml`; expect <N>
- Spot-check any entity row: `curl .../api/states/<entity_id>`; 404 = stale row
```

If a check disagrees with the map, trust the live output, fix the map, and
re-date the section. That loop is the whole maintenance burden: a few
minutes, whenever a check fails.

## What NOT to put in it

- The token, MQTT passwords, or any credential. The map gets pasted into
  agent context freely; it must be safe to leak.
- Full automation YAML (it lives in `automations.yaml`; the map holds the
  index and the gotchas).
- Anything the agent can derive fresh more cheaply than you can maintain it
  (full state dumps, attribute lists).
