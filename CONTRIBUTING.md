# Contributing to OpenClaw Nostr

We use the **Ralph Wiggum** approach for long-running AI agent development. This lets agents ship code autonomously while humans sleep! 😴🚀

## The Ralph Wiggum Approach

> *"I'm helping!"* — Ralph Wiggum

### Core Concept

Run a coding agent in a loop until a feature is complete. Each iteration:
1. Works on ONE single feature
2. Commits progress
3. Checks if complete (`<promise>COMPLETE</promise>`)
4. Exits early if done

### Prerequisites

- tmux (for non-blocking sessions)
- A coding agent (Claude Code, OpenCode, Codex, etc.)
- Git configured

### Setup

**1. Create a tmux session:**
```bash
tmux new-session -d -s ralph-agent
tmux send-keys -t ralph-agent "cd /path/to/openclaw-nostr && ./scripts/ralph.sh --feature 'your-feature-name'" Enter
```

**2. Detach and let it run:**
```bash
tmux detach -s ralph-agent
# Agent continues in background!
```

**3. Check progress anytime:**
```bash
tmux attach -t ralph-agent
# Or just check the logs:
tail -f progress.txt
```

### The Ralph Script

Create `scripts/ralph.sh`:

```bash
#!/bin/bash

FEATURE=$1
MAX_RUNS=${2:-10}
PRD_FILE=".ralph/prd-${FEATURE}.json"

echo "🦞 Ralph starting work on: $FEATURE"

for i in $(seq 1 $MAX_RUNS); do
    echo "--- Run $i/$MAX_RUNS ---" >> progress.txt
    
    # Run coding agent
    claude code \
        --prompt "Work on this single feature: $FEATURE" \
        --output result.txt
    
    # Check for completion
    if grep -q "<promise>COMPLETE</promise>" result.txt; then
        echo "✅ Feature complete!"
        tmux kill-session -t ralph-agent
        exit 0
    fi
    
    sleep 30
done

echo "⚠️ Max runs reached. Review progress.txt"
```

### PRD Format

Create `.ralph/prd-your-feature.json`:

```json
{
  "feature": "Implement NIP-01 agent identity",
  "stories": [
    {
      "id": 1,
      "description": "Create npub/nsec keypair generation",
      "passes": false
    },
    {
      "id": 2,
      "description": "Implement key storage utilities",
      "passes": false
    }
  ]
}
```

### Monitoring

```bash
# Watch progress
watch -n 10 'tail -20 progress.txt'

# Check git log
watch -n 30 'git log --oneline -5'
```

### Rules

1. **One feature at a time** - Scope small!
2. **CI must stay green** - Tests/types pass every commit
3. **Commit often** - Each run = one commit minimum
4. **Update PRD** - Mark stories as passing
5. **Max 10 runs** - Prevents infinite loops

### Example Workflow

```bash
# 1. Pick a small feature from issues
git checkout -b feat/keypair-gen

# 2. Create PRD
mkdir -p .ralph
cat > .ralph/prd-keypair.json << 'JSON'
{
  "feature": "Implement keypair generation",
  "stories": [
    {"id": 1, "description": "Generate random nsec", "passes": false}
  ]
}
JSON

# 3. Start Ralph
tmux new-session -d -s ralph-keypair
./scripts/ralph.sh --feature keypair --max-runs 5

# 4. Go to sleep! 😴

# 5. Next morning: Check progress
tail progress.txt
# Should see: <promise>COMPLETE</promise>
```

## Contributing Without Ralph

That's fine too! Just follow our [issues](https://github.com/vnnkl/openclaw-nostr/issues) and submit PRs traditionally.

---
*Ralph on, friends!* 🦞
