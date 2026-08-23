# Secrets hygiene for agent-driven ops

These skills have an agent SSH into the thing that controls your house and hold
an admin token for it. The agent needs the *capability*. It never needs the
*credential*. Those are different things, and most setups hand over both
without noticing.

An agent transcript is a log file. Anything the agent reads or prints lands in
it, gets summarized into future context, and may sync to whatever service hosts
the session. A token that appears in a transcript once is not a secret anymore.

## The exposure ladder

From worst to best. Each rung removes a place the token can leak.

1. **Token pasted into the chat.** Never. It is now in the transcript forever,
   and in every summary of that transcript.
2. **Token in a file the agent opens.** The common default, and barely better:
   `cat ~/.config/ha/env` puts the token in context just as surely as pasting
   it. `secrets.yaml` counts too; the recipes here never tell the agent to
   read it.
3. **Token in the environment, injected by the shell.** The recipes in these
   skills are written as `-H "Authorization: Bearer $HA_TOKEN"` on purpose:
   the shell expands the variable, the agent composes the command without
   ever seeing the value. This is the floor these skills assume, and it costs
   nothing.
4. **Token held by a broker the agent cannot read.** The credential lives with
   a separate user or host; the agent calls a wrapper, the wrapper injects,
   the agent gets output only. Even a compromised or confused agent has
   nothing to exfiltrate. [HomelabHero](https://github.com/serversathome/homelabhero)
   productized this shape for SSH-driven homelab ops and deserves the credit.

Rung 3 is the sane default for a home instance. Reach for rung 4 when the
agent runs unattended, or against an instance you answer for but do not own.

## Rules that hold at any rung

- **Env file outside any repo, `chmod 600`, never committed.** No repo
  history, no accidental `git add`.
- **Mint the agent its own token under its own name.** Long-lived tokens are
  listed and revocable per token on the profile page, and the token name is
  how you answer "what changed this automation" without guessing. Revoke it
  the day the experiment ends.
- **A long-lived token is full admin.** There is no read-only HA token, which
  makes the conservative doctrine in these skills (read before write, validate
  before reload, before/after reads) your actual guardrail, not the token.
- **Never echo the variable.** Not to debug, not to "check it is set". Use
  `test -n "$HA_TOKEN" && echo set` if you must confirm existence.
- **Watch your own logs.** Shell history, `curl -v`, and pasted debug output
  can all reproduce a header that carries the token. Redact before anything
  leaves the machine.

## The test

Ask the agent what the token is. The right answer is that it cannot know.
If it can answer, you are on rung 2, and one bad prompt away from rung 1.
