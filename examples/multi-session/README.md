# Multi-Session Coordination Demo

Watch two Claude Code sessions coordinate on the same repo via campfire.

## Setup

```bash
# In your project directory:
cf swarm start --description "multi-session demo"
```

## Run

```bash
# Terminal 1 (Session A):
/cf-coordinate start
# → Announces: "Session started. Working on: auth middleware refactor."
# → Reads: no other sessions active

# Terminal 2 (Session B):
/cf-coordinate start
# → Announces: "Session started. Working on: billing endpoint."
# → Reads: "Session A is working on auth middleware refactor."

# Session A changes an interface:
/cf-coordinate schema-change "Changed AuthToken struct: added RefreshToken field. Affects: pkg/billing/client.go"

# Session B checks before continuing:
/cf-coordinate status
# → Sees: "Schema change: AuthToken struct changed. Affects: pkg/billing/client.go"
# → Pulls latest, updates billing code to use new AuthToken shape

# Session B needs a human decision:
/cf-coordinate needs-human "Billing endpoint currently returns 200 on failed charge. Spec says 402. Change will break mobile app v2.1. Proceed?"
```

## Human monitoring

```bash
# In a separate terminal:
cf read "$(cat .campfire/root)" --follow --tag gate-human
# → Sees Session B's question about the billing endpoint
```

## What to observe

1. Sessions discover each other via the root campfire
2. Schema changes are visible to all sessions immediately
3. Human decisions route to a single monitoring point
4. No central orchestrator — each session is independent but aware
