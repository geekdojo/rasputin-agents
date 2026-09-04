---
version: "0.1.2"
level: pair
processes:
  design: pair
  documentation: pair
  review: assist
---

This format is based on [AI-DECLARATION.md](https://ai-declaration.md/en/0.1.2/).

Long-form context — approach, human accountability, provenance, and the rules for
AI-assisted contributions — is in [AI_DISCLOSURE.md](AI_DISCLOSURE.md).

## Notes

- This repository packages an agent skill — `SKILL.md`, the plugin manifests, and docs. It
  carries no production code, no test suite and no CI, so `implementation`, `testing` and
  `deployment` are omitted and implicitly `none`; what it holds is documentation and the
  design decisions behind it.
