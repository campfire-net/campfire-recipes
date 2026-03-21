# Standing Architect Demo

Watch a standing architect answer worker questions in real-time via campfire.

## Setup

```bash
# Terminal 1: Create the campfire
cf init  # if not done already
cf create --description "architect-demo" --json | jq -r .id > /tmp/demo-campfire
CAMPFIRE=$(cat /tmp/demo-campfire)
echo "Campfire: $CAMPFIRE"
```

## Run

```bash
# Terminal 1: Start the architect
# In Claude Code:
/cf-architect "The system uses JWT tokens with 15-minute expiry. Auth middleware validates on every request. Tokens are stored in httpOnly cookies, never localStorage."

# Terminal 2: Worker asks a question
# In Claude Code:
/cf-escalate "Should the token refresh endpoint require the old (expired) token, or use a separate refresh token?"

# Watch Terminal 1: architect wakes up, reads the question, posts a ruling
# Watch Terminal 2: worker's cf await unblocks, receives the ruling
```

## What to observe

1. The architect is dormant (zero output tokens) until a question arrives
2. The worker blocks — its context is preserved, no polling
3. The ruling cites the design context ("JWT with 15-minute expiry means...")
4. After answering, the architect goes dormant again
5. Ask another question — the architect checks its prior rulings for consistency
