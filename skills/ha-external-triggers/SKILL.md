---
name: ha-external-triggers
description: >-
  Use when a Home Assistant automation's "if" is not a device: react to a
  utility's real-time electricity price (load shedding, price-spike alerts,
  demand response), or use an LLM as a sensor via ai_task and a camera
  snapshot for judgment calls (person vs cat vs shadow, "is the garage door
  actually open in this image"). Covers the save/bump/shed/restore
  thermostat loop with hysteresis, threshold helpers, the armed-sentinel
  restore guard, ai_task.generate_data with structured output and camera
  attachments, and branching automations on LLM fields. Not for basic
  automation mechanics (ha-automations) or connecting (ha-connect).
compatibility: Home Assistant 2024.6+ for the ai_task pattern (needs an AI Task entity from an LLM integration). Price pattern needs any integration exposing a price sensor.
---

# HA External Triggers

The trigger doesn't have to be a device. Two patterns where the "if" comes
from outside the house: a market price, and a model's judgment. Both build
on the loop and verify doctrine from `ha-automations` — these are the
automations where silent failure is most expensive, because you'll only
notice on the electric bill or the missed intruder.

## Pattern A: price-driven load shedding

If your utility exposes real-time or hourly pricing (mine does; yours may be
a different utility or a dynamic tariff like Octopus Agile — anything that
lands in HA as a price sensor works), you can automate the response to price
spikes: alert, pre-cool less aggressively, and shut down discretionary load
until the price falls.

### The pieces

```yaml
# Helpers (configuration.yaml or the UI). No `initial:` — tuned values and
# the saved setpoint must survive a restart.
input_number:
  price_alert_threshold:
    min: 5
    max: 60
    step: 1
    unit_of_measurement: "¢/kWh"
  price_shed_threshold:
    min: 5
    max: 60
    step: 1
  pre_spike_setpoint:
    min: 15        # one notch BELOW any real setpoint = "disarmed" sentinel
    max: 30
    step: 0.5
```

`pre_spike_setpoint` is both storage and state machine: at its minimum it
means "no shed in progress" (disarmed); any higher value is the saved
setpoint to restore. This survives restarts, which a variable in an
automation would not.

### Engage: save → bump → shed

```yaml
- id: load_shed_on_price_spike
  alias: Load shed on price spike
  triggers:
    - trigger: numeric_state
      entity_id: sensor.utility_hourly_price
      above: input_number.price_shed_threshold
      for: "00:15:00"          # hysteresis: spikes that last, not blips
  conditions:
    - condition: state
      entity_id: climate.thermostat
      state: cool              # only bump a thermostat that's cooling
    - condition: numeric_state # don't re-engage over an active shed
      entity_id: input_number.pre_spike_setpoint
      below: 16
  actions:
    - action: input_number.set_value        # 1. SAVE the current setpoint
      target: {entity_id: input_number.pre_spike_setpoint}
      data:
        value: "{{ state_attr('climate.thermostat', 'temperature') }}"
    - action: climate.set_temperature       # 2. BUMP cooling, capped
      target: {entity_id: climate.thermostat}
      data:
        temperature: >-
          {{ [ state_attr('climate.thermostat', 'temperature') | float + 2,
               26 ] | min }}
    - action: media_player.turn_off         # 3. SHED discretionary load
      target: {entity_id: media_player.tv}
    # ...any other high-draw, low-regret loads
  mode: single
```

### Release: restore, guarded

```yaml
- id: load_shed_release
  alias: Load shed release
  triggers:
    - trigger: numeric_state
      entity_id: sensor.utility_hourly_price
      # release BELOW threshold minus a margin — never at the same value the
      # engage fires at, or the two automations oscillate at the boundary
      below: "{{ states('input_number.price_shed_threshold') | float - 2 }}"
      for: "00:15:00"
  conditions:
    - condition: numeric_state               # armed sentinel: a shed is active
      entity_id: input_number.pre_spike_setpoint
      above: 16
    - condition: template                    # only restore DOWNWARD; if a
      # human already lowered it past the saved value, leave their choice
      value_template: >-
        {{ state_attr('climate.thermostat', 'temperature') | float >
           states('input_number.pre_spike_setpoint') | float }}
  actions:
    - action: climate.set_temperature
      target: {entity_id: climate.thermostat}
      data:
        temperature: "{{ states('input_number.pre_spike_setpoint') | float }}"
    - action: input_number.set_value         # disarm the sentinel
      target: {entity_id: input_number.pre_spike_setpoint}
      data: {value: 15}
  mode: single
```

(If a `numeric_state` template trigger fights your HA version, put the
margin check in a template trigger or condition instead — the *rule* is what
matters: release threshold < engage threshold.)

### The doctrine, learned the annoying way

- **Hysteresis on both edges.** `for:` on engage and release; a margin
  between the two thresholds. Price feeds jitter and 5-minute prices cross
  thresholds constantly.
- **Save/restore through a helper, guarded by a sentinel.** Restoring
  blindly overwrites human adjustments made mid-shed; restoring without the
  armed check "restores" a setpoint that was never saved.
- **Shed is off-only.** Turn things off on the spike; on release, restore
  the *thermostat* and nothing else. Re-energizing loads automatically (TV
  pops on at 2am when the price drops) is how a clever automation becomes a
  banned one.
- **Stay out of other automations' lanes.** If something else also manages
  the thermostat (eco-when-away, a schedule), add conditions so shed and
  release only act when those aren't in control — two automations fighting
  over one setpoint is undebuggable from the history graph.
- **Alert separately from acting.** A push notification when the price
  crosses the alert threshold (lower than the shed threshold) keeps the
  human in the loop; clear it silently when the price falls rather than
  sending a second notification.

## Pattern B: the LLM as a sensor

Cameras give you `person detected`. They don't give you *"is that a person,
or the cat on the fence, or a headlight shadow?"* — the judgment call that
decides whether a 3am notification is warranted. `ai_task.generate_data`
(HA's AI Task entity, backed by whatever LLM integration you run) takes a
camera snapshot and returns **structured** fields your automation can branch
on.

```yaml
- id: overnight_yard_classify
  alias: Overnight yard motion, LLM-classified
  triggers:
    - trigger: state
      entity_id: binary_sensor.backyard_motion
      to: "on"
  conditions:
    - condition: time
      after: "23:00:00"
      before: "05:00:00"
  actions:
    - action: ai_task.generate_data
      response_variable: ai
      data:
        task_name: yard_scene_classify
        entity_id: ai_task.your_llm_ai_task
        instructions: >-
          This is a backyard security camera frame captured after a motion
          event at night. If the scene is empty or shows only weather,
          shadows, or lighting changes, say so plainly with threat_level
          low. Otherwise describe what is present. Classify threat_level:
          low (empty scene, animals, blowing debris), medium (unexpected
          object or unclear figure), high (person present).
        structure:
          description:
            description: One sentence describing the scene
            selector:
              text:
          threat_level:
            description: low, medium, or high
            selector:
              select:
                options: ["low", "medium", "high"]
        attachments:
          - media_content_id: media-source://camera/camera.backyard
            media_content_type: image/jpeg
    - choose:
        - conditions: "{{ ai.data.threat_level == 'high' }}"
          sequence:
            - action: notify.mobile_app_your_phone
              data:
                title: "Person in the backyard"
                message: "{{ ai.data.description }}"
        - conditions: "{{ ai.data.threat_level == 'medium' }}"
          sequence:
            - action: notify.mobile_app_your_phone
              data:
                title: "Backyard motion"
                message: "{{ ai.data.description }}"
      # low: do nothing. The whole point is the notifications you DON'T get.
  mode: single
```

Fields come back typed per their selector (`text` → string, `select` → one
of the options, `boolean`/`number` likewise) at `ai.data.<field>` — no JSON
parsing, no regex over prose.

### Test it without waiting for a prowler

```bash
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "task_name": "test",
    "entity_id": "ai_task.your_llm_ai_task",
    "instructions": "Describe this image in one sentence and classify threat_level as low, medium, or high.",
    "structure": {
      "description": {"description": "one sentence", "selector": {"text": {}}},
      "threat_level": {"description": "low/medium/high", "selector": {"select": {"options": ["low","medium","high"]}}}
    },
    "attachments": [{"media_content_id": "media-source://camera/camera.backyard", "media_content_type": "image/jpeg"}]
  }' \
  "http://$HA_HOST:8123/api/services/ai_task/generate_data?return_response"
```

Run it in daylight, at night, and against a deliberately boring frame before
trusting it at 3am.

### The doctrine

- **Instruct the empty scene explicitly.** Vision models pattern-match hard;
  shown a dark empty yard and asked "what's the threat", some will find one.
  The prompt must give the model an explicit, honorable exit ("if the scene
  is empty... say so, threat_level low") *before* the classification ask.
- **The LLM is the judgment, not the trigger.** Cheap deterministic signals
  (motion sensor, time window) decide *when* to ask; the model only answers
  the question code can't. Never poll an LLM on a timer to "check" a camera —
  cost, latency, and hallucination all scale with pointless calls.
- **Constrain outputs with selectors**, then branch only on the constrained
  field. Branching on prose (`'person' in ai.data.description`) reintroduces
  everything `structure` exists to prevent.
- **Fail toward the cheap side.** If the ai_task call errors (LLM down,
  quota), decide per-automation what the default branch is — for a security
  notify, fall back to notifying; for a comfort tweak, fall back to nothing.
- **Judgment calls only.** "Is the sun up" is an astronomical fact
  (`sun.sun`), not a vision task. If a template or a sensor can answer,
  don't spend a model call — the LLM-as-sensor pattern earns its keep
  exactly where deterministic signals run out.

## Where else this goes

The same two shapes cover most "external if" ideas: a **feed** (pollen
index, grid CO2 intensity, surf report) driving thresholds with hysteresis
and off-only actions, or a **judgment** (package on the porch? grill left
on? garage door actually closed?) driving a snapshot + `structure` + one
`choose:` block. Build either on top of `ha-automations`' verify loop and
they stay debuggable.
