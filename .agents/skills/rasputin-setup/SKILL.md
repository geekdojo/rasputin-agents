---
name: rasputin-setup
description: Install and set up a Rasputin homelab cluster — flash the first node (Raspberry Pi 4/5/CM5 or Intel N100/amd64) using the non-interactive bootstrap flow, verify checksums and signatures against the Rasputin root CA, confirm the control plane came up via its health probe, and troubleshoot first-boot issues. Use when the user wants to install Rasputin, set up a Rasputin cluster or node, flash a Rasputin image, add a node, or fix a Rasputin first-boot problem (expired certificate, rasputin.local not resolving, seed file not read).
---

# Rasputin setup

<!-- Single canonical copy (github.com/geekdojo/rasputin-agents,
     .agents/skills/rasputin-setup/SKILL.md): Codex discovers it here by
     convention, and the Claude Code plugin manifest points its `skills`
     field at this same directory. -->

Rasputin is an open-source homelab cluster system (AGPL-3.0, pre-alpha): flash a card,
boot, open `http://rasputin.local`. You are driving a real disk-flashing install, so the
guardrails below are part of the contract, not suggestions.

## 0. Fetch current facts first

Versions, URLs, and this contract change — the product is pre-alpha. Before planning,
fetch and prefer over anything you remember:

- https://rasputin.geekdojo.com/llms.txt — index: current stable version, docs, manifests
- https://rasputin.geekdojo.com/docs/agents/index.md — the full install contract (raw markdown)
- https://rasputin.geekdojo.com/releases.json — latest stable versions + image URLs

Authoritative checksums per release:
`https://github.com/geekdojo/rasputin-os/releases/latest/download/manifest.json`
(and the same path on `rasputin-openwrt-firewall`).

## 1. Plan with the user

Establish before touching anything:

1. **First-node hardware** — Raspberry Pi 4/5/CM5 → `RASPUTIN_ARCH=arm64`; Intel N100 or
   any amd64 mini-PC → `RASPUTIN_ARCH=amd64`. This node becomes the control plane.
2. **Boot media** — microSD/NVMe/USB, plugged into the machine you're running on.
3. **SSH public key** — check `~/.ssh/*.pub`. Optional but strongly recommended: images
   bake no key; the seed's key is the only SSH way in.
4. **Network** — wired ethernet with DHCP, IPv4. **A passkey device** for sign-in later.

The flashing host must be macOS or Linux (`bootstrap.sh` exits on anything else).

## 2. Flash — dry-run first, always

```sh
curl -fsSL https://rasputin.geekdojo.com/bootstrap.sh | sudo \
  RASPUTIN_ARCH=<arm64|amd64> RASPUTIN_NODE_ID=cp-1 \
  RASPUTIN_SSH_KEY_FILE=<path/to/key.pub> \
  RASPUTIN_DRY_RUN=1 bash
```

Show the user the resolved plan (disk, image URL, version). **Only after they explicitly
confirm the target disk**, rerun with `RASPUTIN_DRY_RUN=1` replaced by
`RASPUTIN_DISK=<device> RASPUTIN_ASSUME_YES=1`.

Rules:

- Never skip the dry run. Never choose a disk for the user.
- Never set `RASPUTIN_ALLOW_INTERNAL=1` — external media only, unless the user insists
  and states they understand it can destroy their system disk.
- The script verifies SHA-256 against the release manifest and reads the seed back from
  the media; trust its checks, don't re-flash on the first hiccup — read its error.

Manual path (no script): download from `releases.json`, verify `imageSha256` from the
manifest, `xz -d` + write, then place a seed file on the FAT volume **labeled
`RASPUTIN-OS`** — template at https://rasputin.geekdojo.com/rasputin-seed.env.example
(keep the SSH key double-quoted; LF line endings). Signature verification commands (root
CA at https://rasputin.geekdojo.com/rasputin-root-ca.pem) are in the agents doc — note
the firewall artifacts ship detached CMS `.sig`s, OS `.img.xz` is checksum-only.

## 3. Boot and verify — machine-checkable

Slot the media, wired ethernet, power on. First boot takes a few minutes; connection
refused during it is normal. Poll:

```sh
curl -fsS --max-time 5 http://rasputin.local/healthz
# → {"status":"ok"} when the control plane is up
```

If `rasputin.local` never resolves (no mDNS on the client, some routers): find the DHCP
lease for host `rasputin` and probe `http://<ip>/healthz` instead. The cluster CA is
fetchable at `http://rasputin.local/mesh-ca.pem` if the user wants it in their trust
store before the browser step.

## 4. Hand off to the human (required)

Two steps are deliberately human-only — walk the user to a browser and wait:

1. **Trust + passkey**: `http://rasputin.local` lands on a trust page (per-OS CA install,
   with a proceed-past-warning escape hatch), then `https://rasputin.local/setup`
   registers a passkey (Touch ID / Windows Hello / security key). No passwords exist;
   there is nothing for you to type.
2. **Dashboard setup wizard** (banner): name the installation, Finish. Re-runnable.

## 5. More nodes

Use the dashboard's **+** (Add node) wizard — it emits a one-liner with an id-bound join
token baked in. Run that one-liner as given for each new node's media. Do **not**
hand-construct compute seeds (`RASPUTIN_NATS_URL`, `RASPUTIN_CP_JOIN_TOKEN`). Nodes go
PENDING → ONLINE on the hex grid; cluster caps at 24 nodes. The optional firewall node is
a separate x86-only image with its own seed — see the agents doc.

## 6. Troubleshooting

| Symptom | Fix |
| --- | --- |
| Browser: certificate **expired** on a fresh node | Clock is wrong (Pi 5 has no RTC; NTP broken). See https://rasputin.geekdojo.com/docs/provisioning/#time-sync |
| `rasputin.local` won't resolve | Client lacks mDNS or router blocks it — use the DHCP-lease IP for host `rasputin` |
| First boot waits forever / seed ignored | Seed must be named `rasputin-seed.env`, at the root of the FAT volume **labeled** `RASPUTIN-OS` (the Pi image has several FAT partitions — go by label); SSH key double-quoted; LF endings |
| Etcher on macOS: "validation failed" at the end | Usually benign — macOS auto-mounted the seed volume and dirtied it. Check the seed file landed; carry on |
| Anything took over an hour | That's a bug by the project's own definition — file it: https://github.com/geekdojo/rasputin-control-plane/issues |

Pre-alpha honesty: don't put it in front of a network the user cares about yet, and
re-fetch the docs rather than trusting cached details.
