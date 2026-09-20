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
(and the same path on `rasputin-openwrt-firewall`). Each is signed: the detached
signature is the same URL with `.sig` appended, and §2 covers what to do with it.

## 1. Plan with the user

Establish before touching anything:

1. **First-node hardware** — Raspberry Pi 4/5/CM5 → `RASPUTIN_ARCH=arm64`; Intel N100 or
   any amd64 mini-PC → `RASPUTIN_ARCH=amd64`. This node becomes the control plane.
2. **Boot media** — microSD/NVMe/USB, plugged into the machine you're running on.
3. **SSH public key** — check `~/.ssh/*.pub`. Optional but strongly recommended: images
   bake no key; the seed's key is the only SSH way in.
4. **Network** — wired ethernet with DHCP, IPv4. **A passkey device** for sign-in later.

The flashing host must be macOS or Linux (`bootstrap.sh` exits on anything else).

## 2. Fetch the release manifest — and hand it to the flasher

`bootstrap.sh` verifies the release itself: it pins the Rasputin root CA by a SHA-256
fingerprint baked into the script, requires `manifest.json.sig` to verify against that
root, requires the signer to be authorized for OS and firmware images, and only then
reads the image's SHA-256 out of the manifest. It refuses to write anything if any of
that fails, including on a release old enough to have no signature.

So **do not run your own verification of a separate copy.** Fetch the manifest once and
hand that exact file over; the flasher verifies the copy it is about to use. Two copies
fetched independently means the document you checked and the document that decided what
got flashed need not be the same one.

```sh
mkdir -p ./rasputin-release
curl -fsSL -o ./rasputin-release/manifest.json \
  https://github.com/geekdojo/rasputin-os/releases/latest/download/manifest.json
curl -fsSL -o ./rasputin-release/manifest.json.sig \
  https://github.com/geekdojo/rasputin-os/releases/latest/download/manifest.json.sig
```

Pass `RASPUTIN_MANIFEST_FILE=./rasputin-release/manifest.json` on both the dry run and
the real run below (the `.sig` is picked up from alongside it). If either download fails,
just leave `RASPUTIN_MANIFEST_FILE` out — the flasher fetches and verifies its own copy.

The dry run in §3 performs the verification with nothing written, so it is where a
signature problem surfaces. Read what it prints:

- `Release signature verified (signed by …)` — good, carry on.
- anything else — **stop**. Do not flash, and tell the user exactly what it said. A
  failed verification is never something to work around, and there is no flag that
  skips it.
- `this release's signing certificate expired on …` — the release is too old to install
  rather than tampered with. Use the current release (drop any pinned `RASPUTIN_RELEASE`).

## 3. Flash — dry-run first, always

```sh
curl -fsSL https://rasputin.geekdojo.com/bootstrap.sh | sudo \
  RASPUTIN_ARCH=<arm64|amd64> RASPUTIN_NODE_ID=cp-1 \
  RASPUTIN_SSH_KEY_FILE=<path/to/key.pub> \
  RASPUTIN_MANIFEST_FILE=<abs/path/to/manifest.json> \
  RASPUTIN_DRY_RUN=1 bash
```

Show the user the resolved plan (disk, image URL, version) **and the signature line**.
Only after they explicitly confirm the target disk, rerun with `RASPUTIN_DRY_RUN=1`
replaced by `RASPUTIN_DISK=<device> RASPUTIN_ASSUME_YES=1`, keeping
`RASPUTIN_MANIFEST_FILE` so the run flashes the release the dry run verified.

Rules:

- Never skip the dry run. Never choose a disk for the user.
- Never set `RASPUTIN_ALLOW_INTERNAL=1` — external media only, unless the user insists
  and states they understand it can destroy their system disk.
- The script verifies the manifest's signature, checks the image's SHA-256 against that
  verified manifest, and reads the seed back from the media; trust its checks, don't
  re-flash on the first hiccup — read its error.
- `RASPUTIN_MANIFEST_FILE` must point at a path readable by **root** — the script runs
  under `sudo` — so prefer an absolute path.

Manual path (no script — e.g. Raspberry Pi Imager or Etcher): still verify first by
running the dry run above, which checks the signature and prints the image URL and its
`imageSha256`; then download, check that SHA-256 yourself, `xz -d` + write, and place a
seed file on the FAT volume **labeled `RASPUTIN-OS`** — template at
https://rasputin.geekdojo.com/rasputin-seed.env.example (keep the SSH key
double-quoted; LF line endings). Don't hand-roll a verification command: the release's
authorization rules live in the flasher, and a hand-written `openssl cms -verify` checks
the chain but not that the signer is allowed to sign firmware. Firewall artifacts ship
detached CMS `.sig`s of their own; the agents doc has those commands.

## 4. Boot and verify — machine-checkable

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

## 5. Hand off to the human (required)

Two steps are deliberately human-only — walk the user to a browser and wait:

1. **Trust + passkey**: `http://rasputin.local` lands on a trust page (per-OS CA install,
   with a proceed-past-warning escape hatch), then `https://rasputin.local/setup`
   registers a passkey (Touch ID / Windows Hello / security key). No passwords exist;
   there is nothing for you to type.
2. **Dashboard setup wizard** (banner): name the installation, Finish. Re-runnable.

## 6. More nodes

Use the dashboard's **+** (Add node) wizard — it emits a one-liner with an id-bound join
token baked in. Run that one-liner as given for each new node's media. Do **not**
hand-construct compute seeds (`RASPUTIN_NATS_URL`, `RASPUTIN_CP_JOIN_TOKEN`). Nodes go
PENDING → ONLINE on the hex grid; cluster caps at 24 nodes. The optional firewall node is
a separate x86-only image with its own seed — see the agents doc.

## 7. Troubleshooting

| Symptom | Fix |
| --- | --- |
| Browser: certificate **expired** on a fresh node | Clock is wrong (Pi 5 has no RTC; NTP broken). See https://rasputin.geekdojo.com/docs/provisioning/#time-sync |
| `rasputin.local` won't resolve | Client lacks mDNS or router blocks it — use the DHCP-lease IP for host `rasputin` |
| First boot waits forever / seed ignored | Seed must be named `rasputin-seed.env`, at the root of the FAT volume **labeled** `RASPUTIN-OS` (the Pi image has several FAT partitions — go by label); SSH key double-quoted; LF endings |
| Etcher on macOS: "validation failed" at the end | Usually benign — macOS auto-mounted the seed volume and dirtied it. Check the seed file landed; carry on |
| Anything took over an hour | That's a bug by the project's own definition — file it: https://github.com/geekdojo/rasputin-control-plane/issues |

Pre-alpha honesty: don't put it in front of a network the user cares about yet, and
re-fetch the docs rather than trusting cached details.
