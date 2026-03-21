# Escalation Cascade Demo

Watch a message posted at a leaf campfire propagate to the root via tag filtering.

## Setup

```bash
cf init  # if not done already

# Create 3-level hierarchy
ROOT=$(cf create --description "project-root" --json | jq -r .id)
TEAM=$(cf create --description "team-alpha" --require gate-human --require schema-change --json | jq -r .id)
# Team campfire joins root — messages with matching tags propagate up
# (In practice, cf join handles the nesting)

echo "Root: $ROOT"
echo "Team: $TEAM"
```

## Run

```bash
# Terminal 1: Human monitors the root
cf read "$ROOT" --follow --tag gate-human

# Terminal 2: Worker posts a question to the team campfire
cf send "$TEAM" --instance worker --tag gate-human \
  "Need approval: migration will drop the legacy_users table. 3 integrations still reference it."

# Watch Terminal 1: message appears at the root
# The human never joined the team campfire — the message propagated via --require
```

## What to observe

1. The `--require gate-human` on the team campfire means any message tagged `gate-human` propagates to the parent
2. The human monitors one campfire (root) and sees decisions from all teams
3. Schema changes propagate the same way — `--require schema-change`
4. Add more team campfires under the same root — the human's monitoring command doesn't change
