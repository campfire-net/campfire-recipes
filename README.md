# Campfire Recipes

Ready-to-use coordination patterns for AI agents using the [Campfire protocol](https://getcampfire.dev). Skills, agent specs, and examples for Claude Code.

## What's here

| Recipe | What it does |
|--------|-------------|
| **cf-architect** | Standing design authority. Blocks on `cf read --follow`, answers worker questions in real-time. Workers block on `cf await` until answered. |
| **cf-escalate** | One-command escalation. Worker posts a question, blocks until the architect (or human) answers. |
| **cf-coordinate** | Multi-session coordination. Announces what you're working on, reads what others changed, routes human-needed decisions to a single monitoring point. |
| **architect agent** | Agent spec for the standing architect. Drop into `.claude/agents/`. |
| **implementer agent** | Agent spec showing escalation protocol, campfire signals, and merge coordination. |

## Install

```bash
# Clone
git clone https://github.com/campfire-net/campfire-recipes.git

# Copy skills to Claude Code global directory
cp -r campfire-recipes/.claude/skills/* ~/.claude/skills/

# Copy agent specs to your project (optional — adapt to your needs)
cp campfire-recipes/.claude/agents/* your-project/.claude/agents/
```

New Claude Code sessions pick up skills automatically.

## Prerequisites

- [Campfire CLI](https://getcampfire.dev) (`cf`) on PATH
- [Claude Code](https://claude.ai/claude-code)

## The Patterns

### Standing Architect

One long-running agent holds design context. Workers ask it questions instead of guessing.

```
Session 1 (architect):
  /cf-architect "Design doc: docs/design/auth-system.md"
  → Loads design context
  → Blocks on cf read --follow --tag escalation
  → Dormant until someone asks

Session 2 (worker):
  /cf-escalate "Should auth tokens use optimistic or pessimistic locking?"
  → Posts --future to campfire
  → Blocks on cf await
  → Architect wakes, reads question, posts ruling
  → Worker resumes with the answer, full context preserved
```

Cost: the architect loads design context once. Workers never load it. N workers share one architect. The architect is dormant between questions — zero token cost when idle.

### Escalation Cascade

Messages flow up through nested campfires via tag filtering. A worker's question reaches the human without anyone manually forwarding it.

```
Project Root Campfire                    ← Human monitors here
  └── Team Campfire (--require gate-human)
        └── Worker posts --tag gate-human "Need approval for X"
            → Propagates automatically to root
```

The human runs one command to see everything that needs them across all teams:
```bash
cf read "$root" --follow --tag gate-human
```

### Multi-Session Merge Coordination

Multiple agents working the same repo announce what they're touching. Schema changes propagate to all sessions via the root campfire. Merges serialize through PRs.

```
Session A: cf send "$root" --tag status "Working on auth. Touching: pkg/auth/, pkg/middleware/"
Session B: cf send "$root" --tag status "Working on billing. Touching: pkg/billing/"
Session B: cf read "$root" --tag schema-change --peek
  → Sees Session A changed the auth interface
  → Pulls and rebases before continuing
```

## Real-world results

These patterns were developed and field-tested across a portfolio of 11 active projects. One developer, AI agents coordinating via Campfire:

- **1,400+ commits** across 11 repos in one week (Mar 14-21, 2026)
- **300+ features** shipped
- **1,700+ agent dispatches** coordinated through campfire
- **3 concurrent swarms** on the same repo without merge conflicts (after adopting the coordination pattern — 18% merge conflict rate before)

Your mileage will vary. These numbers reflect a specific workflow with specific tools. The coordination patterns are general-purpose.

## Examples

The `examples/` directory contains runnable demos:

- **`standing-architect/`** — Launch an architect + 2 workers. Watch real-time escalation and fulfillment.
- **`escalation-cascade/`** — 3-level campfire hierarchy. Post at the leaf, see it at the root.
- **`multi-session/`** — 2 sessions coordinating on the same repo via root campfire.

## How these were built

These recipes were extracted from [Resonant Product Theory](https://3dl.dev/resonant-software.html) — a methodology for building software with AI agents. The recipes are the coordination primitives. The methodology (measurement, self-improvement, adversarial quality) is separate.

Campfire provides the protocol. These recipes show one way to use it. Build your own.

## License

MIT — same as Campfire.
