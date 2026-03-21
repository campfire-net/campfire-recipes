---
name: cf-architect
description: Launch a standing architect agent that monitors a campfire for escalation questions and answers them using design context. Workers block on cf await until answered.
argument-hint: [design doc path or "Design campfire: <id>"]
---

Launch a standing architect that holds design context and answers worker questions via campfire.

**Input:** $ARGUMENTS

Parse arguments for:
- A file path → read as the design document
- `Design campfire: <id>` → read the design team's deliberation history
- Both → use both

## Setup

1. Find or create a swarm campfire:
   ```bash
   # Use existing swarm root if available:
   campfire=$(cat .campfire/root 2>/dev/null)
   # Or create one:
   [ -z "$campfire" ] && campfire=$(cf create --description "architect session" --json | jq -r .id)
   echo "Architect campfire: $campfire"
   ```

2. Load design context:
   - If a design doc path was provided, read it fully
   - If a design campfire ID was provided: `cf read <id> --all`
   - If neither: warn "No design context. Architect will make rulings based on code only."

3. Announce:
   ```bash
   cf send "$campfire" --instance architect --tag status "Architect online. Monitoring for escalations."
   ```

## Monitor Loop

Block on the campfire waiting for escalation messages:

```bash
cf read "$campfire" --follow --tag escalation --json
```

For each escalation that arrives:

1. Read the escalation message fully.
2. Check prior rulings: `cf read "$campfire" --tag decision --all`
   - If this question was already answered, fulfill immediately citing the prior ruling.
3. If new question:
   - Read any source code referenced in the escalation
   - Make a ruling grounded in the design context
   - Post fulfillment:
     ```bash
     cf send "$campfire" --instance architect --fulfills <msg-id> --tag decision "<ruling with reasoning>"
     ```
4. Check for schema-change drift:
   ```bash
   cf read "$campfire" --tag schema-change --peek
   ```
   If a schema-change contradicts the design, post a warning.

5. Return to blocking on `--follow`.

## Ruling Standards

- **Decide, don't defer.** Workers are blocked. A fast good-enough ruling beats a slow perfect one.
- **Cite the design.** Every ruling references the section of the design doc or campfire that supports it.
- **Be consistent.** Check prior rulings before making new ones.
- If you truly can't decide (novel trade-off, major consequences): escalate to the human:
  ```bash
  cf send "$campfire" --instance architect --tag gate-human "Need human ruling: <question>"
  ```

## Exit

When the user signals done, or when no escalations arrive for 30 minutes:
```bash
cf send "$campfire" --instance architect --tag status "Architect signing off. Rulings made: N."
```
