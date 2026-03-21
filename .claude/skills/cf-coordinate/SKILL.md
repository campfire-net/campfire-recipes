---
name: cf-coordinate
description: Multi-session coordination via campfire. Announce what you're working on, see what others changed, route decisions to a human monitoring point.
argument-hint: start | status | schema-change "<description>" | needs-human "<question>"
---

Coordinate with other sessions working the same repo.

**Input:** $ARGUMENTS

## Commands

### `start` — Join or create the project campfire

```bash
root_cf=$(cat .campfire/root 2>/dev/null)
if [ -z "$root_cf" ]; then
    echo "No project campfire. Creating one..."
    cf swarm start --description "$(basename $(git rev-parse --show-toplevel)) coordination"
    root_cf=$(cat .campfire/root)
fi
echo "Project campfire: $root_cf"

# Announce this session
cf send "$root_cf" --instance session --tag status \
  "Session started. Working on: <describe what you're about to do>."
```

Read what other sessions posted:
```bash
cf read "$root_cf" --tag status --all
cf read "$root_cf" --tag schema-change --peek
```
Report any active sessions and recent schema changes.

### `status` — See what's happening

```bash
root_cf=$(cat .campfire/root)
echo "=== Active sessions ==="
cf read "$root_cf" --tag status --all
echo ""
echo "=== Schema changes ==="
cf read "$root_cf" --tag schema-change --all
echo ""
echo "=== Needs human ==="
cf read "$root_cf" --tag gate-human --peek
```

### `schema-change` — Notify others you changed a shared interface

```bash
root_cf=$(cat .campfire/root)
cf send "$root_cf" --instance session --tag schema-change "$ARGUMENTS"
```

This is visible to all sessions on the project campfire. Other sessions should `git pull` and check for conflicts before continuing work that touches the same interface.

### `needs-human` — Route a decision to the human

```bash
root_cf=$(cat .campfire/root)
cf send "$root_cf" --instance session --tag gate-human "$ARGUMENTS"
```

The human monitors `cf read "$root_cf" --follow --tag gate-human` to see everything that needs them.

## Setup Views (run once per project)

For the human monitoring experience:
```bash
root_cf=$(cat .campfire/root)
cf view create "$root_cf" "needs-me" --predicate '(tag "gate-human")'
cf view create "$root_cf" "changes" --predicate '(tag "schema-change")'
cf view create "$root_cf" "sessions" --predicate '(tag "status")'
```

Then monitor with:
```bash
cf view read "$root_cf" "needs-me"          # What needs you
cf read "$root_cf" --follow --tag gate-human  # Real-time stream
```
