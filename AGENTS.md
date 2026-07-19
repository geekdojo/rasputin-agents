# Rasputin — agent instructions

This repo packages the agent-facing install/ops experience for
[Rasputin](https://rasputin.geekdojo.com), an open-source homelab cluster system
(pre-alpha, AGPL-3.0).

If you're here to help a user **install or run Rasputin**: use the `rasputin-setup`
skill in `.agents/skills/` (auto-discovered by agents supporting the
[Agent Skills](https://agentskills.io) format). It encodes the safe flashing flow
(dry-run first, external disks only), verification, and troubleshooting.

Ground truth is always the live site — fetch, don't trust memory (pre-alpha; things
change):

- https://rasputin.geekdojo.com/llms.txt — index (current stable version, docs, manifests)
- https://rasputin.geekdojo.com/docs/agents/index.md — the full install contract, raw markdown
- https://github.com/geekdojo/rasputin-os/releases/latest/download/manifest.json — authoritative checksums

Repo layout: there is exactly **one** copy of the skill, at
`.agents/skills/rasputin-setup/SKILL.md`. Codex-style agents discover it there by
convention; the Claude Code plugin (repo root = plugin root, manifests in
`.claude-plugin/`) points its `skills` field at the same directory, and
`marketplace.json` makes this repo an installable marketplace. Don't reintroduce a
second copy.

Hard rules for any agent working in this repo: never weaken the dry-run-first /
external-disk-only guardrails in the skill; never claim remote/away-from-home access
works (untested); keep the pre-alpha caveat.
