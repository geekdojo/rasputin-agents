# rasputin-agents

Agent-facing install and ops tooling for [Rasputin](https://rasputin.geekdojo.com) — the
open-source homelab cluster system (pre-alpha, AGPL-3.0). One skill, packaged for the two
big agent ecosystems.

## Claude Code

```
/plugin marketplace add geekdojo/rasputin-agents
/plugin install rasputin@geekdojo
```

Then ask Claude to "install Rasputin on my homelab" — the `rasputin:rasputin-setup`
skill drives the flash (dry-run first, always), verifies checksums and signatures,
polls the cluster's health probe, and hands off the passkey step to you.

## OpenAI Codex (and other Agent Skills clients)

```
git clone https://github.com/geekdojo/rasputin-agents
cd rasputin-agents && codex
```

Codex auto-discovers the skill from `.agents/skills/` and reads `AGENTS.md` — then ask it
to install Rasputin. Any agent supporting the open
[Agent Skills](https://agentskills.io) format works the same way.

## No agent at all

Everything the skill does is plain shell against documented endpoints:
[Install with an AI agent](https://rasputin.geekdojo.com/docs/agents/) is the
human-readable contract, and
[Getting started](https://rasputin.geekdojo.com/docs/getting-started/) is the
ten-minute path.

## Layout

```
.claude-plugin/marketplace.json          this repo as a Claude Code plugin marketplace
plugins/rasputin/                        the "rasputin" plugin (canonical skill lives here)
  .claude-plugin/plugin.json
  skills/rasputin-setup/SKILL.md
.agents/skills/rasputin-setup/SKILL.md   byte-identical copy for Codex discovery
AGENTS.md                                instructions for agents working in this repo
```

The two `SKILL.md` copies must stay identical — CI-free rule for now, enforced by review.

Pre-alpha, honestly labeled: image layouts and update formats still change without
notice. Issues → [rasputin-control-plane](https://github.com/geekdojo/rasputin-control-plane/issues).
