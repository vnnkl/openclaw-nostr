# Recommended Skills for OpenClaw Nostr

This project integrates with several agent skills from [skills.sh](https://skills.sh) to enhance functionality.

## Required Skills

### Nostr Protocol
```bash
npx skills add soapbox-pub/nostr-skills@nostr
```
Provides:
- NIP (Nostr Implementation Possibilities) documentation
- Event kinds reference
- Protocol best practices

### Bitcoin Lightning Payments
```bash
npx skills add getalby/alby-agent-skill@alby-bitcoin-payments-agent-skill
```
Provides:
- NWC (Nostr Wallet Connect) client
- Lightning Tools (BOLT-11, LNURL)
- Testing wallets for development
- Send/receive payments

## Optional Skills

### Find Skills
```bash
npx skills add vercel-labs/skills@find-skills
```
Discover and install additional skills from the ecosystem.

### Nostr Authentication Keys
```bash
npx skills add soapbox-pub/nostr-skills@nak
```
Key generation and management utilities.

## Skill Ecosystem

Browse all available skills at [skills.sh](https://skills.sh)

Search for relevant skills:
```bash
npx skills find nostr      # Nostr-related skills
npx skills find lightning  # Lightning Network skills
npx skills find social     # Social media/agent skills
```

## Development Setup

1. Install required skills:
```bash
npx skills add soapbox-pub/nostr-skills@nostr -g -y
npx skills add getalby/alby-agent-skill@alby-bitcoin-payments-agent-skill -g -y
```

2. The skills are automatically symlinked to your agents (Claude Code, Moltbot, Cursor, OpenCode)

3. Reference skill documentation when implementing features:
   - `~/.agents/skills/nostr/SKILL.md`
   - `~/.agents/skills/alby-bitcoin-payments-agent-skill/SKILL.md`

---
*Skills make agents smarter* 🧠🦞
