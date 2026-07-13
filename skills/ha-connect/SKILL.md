---
name: ha-connect
description: >-
  Use when connecting a coding agent to a Home Assistant instance for the
  first time or in a fresh session: "connect to Home Assistant", "SSH into
  HA", set up the Terminal & SSH add-on, generate a long-lived access token,
  call the REST API, check an entity's state, call a service from the command
  line, edit configuration or automations YAML over SSH, figure out which
  of the two access lanes (SSH vs REST) a task needs, or decide whether an
  MCP integration already covers the task instead. Foundation skill: the
  other ha-* skills assume its connection pattern and doctrine. Not for
  writing automations (ha-automations) or building the entity inventory
  (ha-context-map).
compatibility: Home Assistant OS or Supervised (needs the Terminal & SSH add-on and the ha CLI). REST recipes work against any HA install.
---

# HA Connect

Two ways into a Home Assistant box, and when to use each. Every other `ha-*`
skill assumes this connection pattern, so get this working first.

## Lane zero: check whether MCP already covers the task

Before setting up SSH and a token, ask what the task actually needs. If it is
purely **state reads and service calls** (control a device, answer "is the
garage open", toggle things on a schedule the UI could build), and the user
has Home Assistant's MCP Server integration (or a community MCP server from
HACS) connected to this agent, use that instead: it is scoped to exposed
entities and needs no root access. Recommend it to users who only want
control and Q&A; don't push them into the SSH lane they don't need.

The lanes below earn their setup cost when the task is **admin-shaped**:
editing YAML, validating config before a reload, reading logs to find out why
an automation never fires, auditing `automations.yaml` for conflicts, taking
backups. The MCP surface cannot do any of that; it is state and services
only, not files.

## The two lanes

| Lane | Transport | Use for |
|------|-----------|---------|
| **SSH** | Terminal & SSH add-on, port 22 | Editing YAML in `/config`, the `ha` CLI (`ha core check`, `ha core info`, `ha backups new`), reading logs, anything file-shaped |
| **REST API** | HTTP :8123 + long-lived token | Live entity state, calling services, rendering templates, verification reads: anything state-shaped |

Rule of thumb: **files over SSH, state over REST.** Editing `automations.yaml`
via the API is miserable; reading a sensor over SSH by grepping a database is
worse.

## One-time setup

### SSH lane: the Terminal & SSH add-on

Settings → Add-ons → Add-on Store → **Terminal & SSH** (official). In its
configuration, add your public key under `authorized_keys` (preferred over a
password), enable the add-on, and confirm port 22 is exposed on the host.

```bash
ssh root@homeassistant.local
# or a single command:
ssh root@homeassistant.local 'ha core info'
```

Two things to know about where you land:

- You are in the **add-on container**, not the host OS. `/config` is mounted
  there and is the same `/config` Core sees; that's where
  `configuration.yaml`, `automations.yaml`, and `scripts.yaml` live.
- The `ha` CLI is available and is the supported way to talk to the
  supervisor: `ha core check`, `ha core restart`, `ha addons`, `ha backups new`.

If `homeassistant.local` doesn't resolve (some networks break mDNS), use the
instance's IP or hostname; the recipes below write it as `<HA_HOST>`.

### REST lane: a long-lived access token

In the HA web UI: click your user (bottom of sidebar) → **Security** tab →
**Long-lived access tokens** → Create Token. It is shown **once**.

Store it in an env var or an untracked env file, never in a skill, a script,
a committed file, or a chat transcript:

```bash
# ~/.config/ha/env  (chmod 600, gitignored)
export HA_HOST="homeassistant.local"
export HA_TOKEN="<paste once, here only>"
```

```bash
source ~/.config/ha/env
curl -s -H "Authorization: Bearer $HA_TOKEN" "http://$HA_HOST:8123/api/"
# → {"message":"API running."}
```

That health check is the first command of every session. If it fails, nothing
else in these skills will work; fix the token/host before debugging anything
downstream.

## REST recipes

All assume `source ~/.config/ha/env` has run.

```bash
# Single entity state (the verification primitive; you will use this constantly)
curl -s -H "Authorization: Bearer $HA_TOKEN" \
  "http://$HA_HOST:8123/api/states/light.living_room"

# All states (big; filter it)
curl -s -H "Authorization: Bearer $HA_TOKEN" \
  "http://$HA_HOST:8123/api/states" | jq -r '.[].entity_id' | sort

# Call a service
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "light.living_room"}' \
  "http://$HA_HOST:8123/api/services/light/turn_on"

# List every service a domain offers (when you're not sure of the name)
curl -s -H "Authorization: Bearer $HA_TOKEN" \
  "http://$HA_HOST:8123/api/services" | jq '.[] | select(.domain=="climate")'

# Render a Jinja template against live state (test template logic BEFORE
# putting it in an automation)
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"template": "{{ states(\"sensor.outdoor_temperature\") | float(0) > 25 }}"}' \
  "http://$HA_HOST:8123/api/template"

# Services that return data need ?return_response
curl -s -X POST -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "weather.home", "type": "daily"}' \
  "http://$HA_HOST:8123/api/services/weather/get_forecasts?return_response"
```

Gotchas worth knowing before they bite:

- **A service call returning `[]` or `200` is not proof anything happened.**
  It means HA accepted the call. Read the entity state back afterwards
  (see `ha-automations` for the full verify doctrine).
- **A call naming a nonexistent service or entity can fail quietly** from the
  caller's perspective (or 400 with a terse message). When a call "did
  nothing", first confirm the service exists via `/api/services` and the
  entity id via `/api/states/<id>`; typos are the #1 cause.
- `POST /api/services/...` responds with a list of states that *changed* as a
  direct result. An empty list on a `turn_on` for an already-on light is
  normal, not an error.

## SSH recipes

```bash
# Where the YAML lives
ssh root@<HA_HOST> 'ls -la /config/'

# Validate config BEFORE any reload/restart: the single most important command
ssh root@<HA_HOST> 'ha core check'

# Read the error log when something misbehaves
ssh root@<HA_HOST> 'ha core logs' | tail -50

# Snapshot before surgery
ssh root@<HA_HOST> 'ha backups new --name "pre-automation-edit"'

# Version/context for the session
ssh root@<HA_HOST> 'ha core info; ha os info'
```

Editing files remotely: prefer pulling the file down, editing locally, and
pushing back, or a quoted heredoc, over inline `sed`/`awk` with nested
escaping (which corrupts YAML edits with depressing regularity):

```bash
scp root@<HA_HOST>:/config/automations.yaml ./automations.yaml
# edit locally, then:
scp ./automations.yaml root@<HA_HOST>:/config/automations.yaml
ssh root@<HA_HOST> 'ha core check'
```

Always keep the previous version (`cp automations.yaml automations.yaml.bak`
on the host, or rely on the backup you just took).

## Session doctrine

1. `source` the env file, run the API health check.
2. Read before you write: pull current state / current YAML first.
3. Validate (`ha core check`) before reloading; reload the specific domain
   (see `ha-automations`) instead of restarting Core.
4. After every write, read the thing back and show before/after.
5. Token hygiene: the token grants full admin. Treat it like a root password.
   Never echo it, never commit it, and revoke tokens you stop using (same
   Security tab).

## What next

- Build the instance map so you stop guessing entity ids: `ha-context-map`.
- Write and verify automations: `ha-automations`.
- Drive automations from prices and LLM judgment: `ha-external-triggers`.
