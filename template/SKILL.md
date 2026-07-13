---
name: ha-yourskill
description: >-
  Use when [the specific situations, symptoms, and phrases that should trigger
  this skill]. Pack the words someone would actually type or think when they
  hit this problem. Describe ONLY when to use it, never summarize the workflow
  inside. Keep the whole frontmatter under 1024 characters. End by naming what
  this skill is NOT for, pointing at the sibling skill that covers it.
compatibility: State what the skill needs (HA OS vs any install, minimum Core version, a specific add-on or integration).
---

# HA Yourskill

One or two sentences: what this skill covers and that it assumes the
connection lanes, env pattern, and doctrine from `ha-connect`. Cross-reference
sibling skills by name (`ha-context-map`, `ha-automations`,
`ha-external-triggers`) rather than repeating their content.

## Guidelines for a good HA skill

Delete this section in your real skill; it's guidance for authoring.

- **Name:** `ha-<area>`, lowercase and hyphens only. The directory name MUST
  equal the frontmatter `name`.
- **Description:** starts with "Use when…", lists concrete triggers and
  keywords, ends with a "not for X, use Y" pointer. No workflow summary (an
  agent will follow the description instead of reading the body).
- **Markdown only.** This collection ships doctrine and fenced curl/ssh/YAML
  recipes, not scripts. The agent writes throwaway code per task; recipes
  stay honest and adaptable where scripts rot.
- **Original prose only.** Don't paste from the Home Assistant docs; write
  the lesson the docs don't teach, and link the docs as the canonical manual.
- **Parameterize everything instance-specific.** `<HA_HOST>` or
  `homeassistant.local`, never a real IP; `$HA_TOKEN` from env, never a
  literal; generic entity ids (`light.living_room`, `media_player.tv`,
  `climate.thermostat`), never your actual house.
- **Verify against ground truth.** Every write ends with reading the entity
  or config back and showing before/after. Show the reader how to confirm
  the change, not just make it.
- Keep `SKILL.md` under ~500 lines; push heavy detail into `references/*.md`.

## Overview

What this is and the core principle in 1–2 sentences.

## When to use

Symptoms and situations (bullets). When NOT to use.

## [Your sections]

Quick-reference recipes, one worked example, the specific traps you learned.
One excellent example beats five generic ones.
