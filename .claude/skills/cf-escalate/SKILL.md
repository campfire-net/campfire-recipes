---
name: cf-escalate
description: Escalate a design question via campfire and block until answered. The standing architect (or a human) fulfills the request. Your context is preserved while waiting.
argument-hint: "<question>"
---

Escalate a question and block until answered.

**Input:** $ARGUMENTS

The question to ask. Be specific — include file paths, line numbers, constraints, and trade-offs. The more context you provide, the faster and better the ruling.

## Find the campfire

```bash
# Swarm campfire (preferred):
campfire=$(cat .campfire/root 2>/dev/null)
# Or discover from cf ls:
[ -z "$campfire" ] && campfire=$(cf ls --json 2>/dev/null | python3 -c "
import json,sys
cfs = json.load(sys.stdin)
if cfs: print(cfs[-1]['campfire_id'])
" 2>/dev/null)
```

If no campfire found, tell the user: "No campfire available. Start one with `cf swarm start` or create one with `cf create`."

## Escalate

```bash
# Post the question as a future
msg_id=$(cf send "$campfire" --instance worker --tag escalation --tag architecture --future \
  "$ARGUMENTS" --json | jq -r .id)

echo "Escalation posted: $msg_id"
echo "Waiting for ruling..."

# Block until someone fulfills it
decision=$(cf await "$campfire" "$msg_id" --timeout 15m --json)
```

## Handle response

- **Fulfilled**: Parse the decision. Report it. Continue your work using the ruling.
- **Timeout**: Report "Escalation timed out after 15 minutes. No architect or human responded." Post a blocker:
  ```bash
  cf send "$campfire" --instance worker --tag blocker "Escalation $msg_id timed out. Cannot proceed without ruling."
  ```

## Tips

- Run `/cf-architect` in another session first — it monitors for these escalations.
- If no architect is running, a human can fulfill manually: `cf send <campfire> --fulfills <msg-id> --tag decision "Use approach A because..."`
- The escalation is visible to everyone on the campfire. Anyone with the answer can fulfill it.
