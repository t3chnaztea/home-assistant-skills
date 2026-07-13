---
name: ha-automations
description: >-
  Use when writing, editing, debugging, or verifying Home Assistant
  automations with an agent: "write an automation that...", turn an English
  description into automation YAML, edit automations.yaml over SSH, an
  automation that never fires or fired once and went dead, a service call
  that silently did nothing, testing with automation.trigger, or reloading
  automations without restarting Core. Covers the write → check → reload →
  verify loop and the silent-failure traps (event-entity triggers, dead
  notify services, no-op writes). Not for the initial connection (ha-connect)
  or price/LLM-driven triggers (ha-external-triggers).
compatibility: Home Assistant with UI-managed automations in /config/automations.yaml, reachable per ha-connect.
---

# HA Automations

Describe in English → write YAML → `ha core check` → reload → **verify**.
Home Assistant's failure mode is almost never an error message; it's an
automation that loads fine and silently never does what you meant. The loop
below exists because of that. Assumes the lanes and env from `ha-connect`,
and entity ids confirmed via `ha-context-map` — never guessed.

## The loop

### 1. Pin down the English first

"Turn off the TV when nobody's around" hides every real decision: which
entity is the TV, what does "nobody's around" mean (motion silence? for how
long?), what must *prevent* it (media actively playing?), does it ever turn
back on. Write the trigger / conditions / actions as three English bullets
and get them agreed before YAML. Ambiguity in, dead automation out.

### 2. Write the YAML

UI-managed automations live in `/config/automations.yaml` as a list; each
entry needs a unique `id:` (the UI uses timestamps; any unique string works)
and an `alias:`.

```yaml
- id: tv_idle_shutdown
  alias: TV idle shutdown
  description: Turn off the TV after 2h with no motion in the living areas
  triggers:
    - trigger: state
      entity_id:
        - binary_sensor.living_room_motion
        - binary_sensor.kitchen_motion
      to: "off"
      for: "02:00:00"
  conditions:
    - condition: state
      entity_id: media_player.tv
      state: "on"        # don't fire into a TV that's already off
  actions:
    - action: media_player.turn_off
      target:
        entity_id: media_player.tv
  mode: single
```

Back up before editing, edit via the pull-down/push-up pattern from
`ha-connect` (not inline `sed` over SSH), and know that **the entity id HA
mints for the automation slugs from the `alias`, not the `id`** —
`automation.tv_idle_shutdown` here, but rename the alias and the entity id
follows on reload.

Test any nontrivial Jinja against live state via `POST /api/template`
*before* embedding it (recipe in `ha-connect`).

### 3. Check, then reload — never skip, never restart

```bash
ssh root@<HA_HOST> 'ha core check'          # must pass before reload
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" -d '{}' \
  "http://$HA_HOST:8123/api/services/automation/reload"
```

`ha core check` catches YAML/schema errors; the reload picks up
`automations.yaml` without a Core restart (a restart drops every `for:`
timer currently counting and briefly kills the whole house — reserve it for
`configuration.yaml` changes that actually require it).

### 4. Verify. The step that isn't optional

A clean reload proves the YAML parsed. It does not prove the automation
exists as you intended, will trigger, or that its actions work. Three
verification tiers, cheapest first:

```bash
# (a) The automation entity exists and is on
curl -s -H "Authorization: Bearer $HA_TOKEN" \
  "http://$HA_HOST:8123/api/states/automation.tv_idle_shutdown" \
  | jq '{state, attributes: {last_triggered: .attributes.last_triggered}}'

# (b) Force-run the ACTIONS (skips triggers/conditions) and watch the target
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/states/media_player.tv" | jq .state
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" -H "Content-Type: application/json" \
  -d '{"entity_id": "automation.tv_idle_shutdown"}' \
  "http://$HA_HOST:8123/api/services/automation/trigger"
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/states/media_player.tv" | jq .state

# (c) For the real trigger path: cause the condition (or wait for it), then
# read last_triggered and the trace (UI: Settings → Automations → Traces)
```

Tier (b) is the workhorse: **read the target entity before, trigger, read it
after, and show both values.** "The call returned 200" is not a result;
`"on" → "off"` is. Note `automation.trigger` bypasses conditions by default —
it validates the actions, not the logic.

## The silent-failure traps

These are the ways an automation lies to you. Check them *before* declaring
one done, and first when debugging one that "just stopped working".

- **Writes can silently no-op.** A service call to a wrong-but-plausible
  entity id, a `climate.set_temperature` in the wrong HVAC mode, a
  `turn_off` on something already off — all return success. The only proof
  of a write is re-reading the entity afterwards. Before/after or it didn't
  happen.
- **A dead service call aborts the rest of the automation.** If an action
  calls a service that no longer exists (the classic: a
  `notify.mobile_app_<old_phone>` left behind by a replaced phone), the
  automation errors out at that step and every later action is skipped —
  with nothing visible unless you read the trace or the error log. When an
  automation "partially works", look for a dead action above the missing
  ones.
- **Event-entity attribute triggers go dead after the first event.** A state
  trigger with `attribute: event_type, to: "pressed"` on an `event.*` entity
  only fires when the attribute *changes*. If the entity has a single event
  type (many doorbells), the attribute never changes again and the
  automation dies silently after one firing. Trigger on the entity's state
  with `not_from: ["unknown", "unavailable"]` instead — the state is the
  last-event timestamp, which changes every event. (Event entities that
  clear between detections don't have this problem; know which kind yours
  is.)
- **`for:` durations reset on any flicker** of the watched state, and a Core
  restart (not a reload) resets them all. An automation that "never fires"
  on a long `for:` may be getting reset by a chattering sensor — check the
  sensor's history.
- **Guessed entity ids fail quietly.** `ha core check` does not validate
  that entity ids exist. `light.livingroom` in an action is a runtime
  error in a trace you'll never read, not a config error. Ids come from
  `ha-context-map`, verified, or from a live `/api/states` read.
- **Renaming the alias renames the `automation.*` entity id**, breaking
  anything that referenced the old one (dashboards, other automations).

## Debugging one that never fires

In order: (1) the automation entity is `on` (not `unavailable` — that's an
orphaned registry entry); (2) `last_triggered` — never, or fired-then-dead?
(3) fired-then-dead on an event entity → the attribute-trigger trap above;
(4) never → verify each trigger entity id exists and check the trace of a
forced condition; (5) fires but actions misbehave → dead-service abort, then
per-action before/after reads.

Traces (Settings → Automations → your automation → Traces) store the last
runs with per-step results — read them before theorizing.

## Doctrine

- One automation per concern; `mode: single` unless you have a reason.
- Prefer conditions that make firing *safe over clever* (guard "is it even
  on?" rather than trusting the trigger).
- Off-actions are safer than on-actions to automate; be conservative about
  automations that turn things on or unlock/open anything.
- Every change ends with a before/after read of the affected entity, pasted,
  not asserted.
