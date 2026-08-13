# skills

Portable Agent Skills for tonia. The official skill is **`tonia-sdk`**.

It walks a coding agent from issuing a `tonia_sk_…` key in the portal,
through Cursor / Claude Code / Codex, to `@tonia/sdk` / `tonia` app code.

## Install

- Portable: `gh skill install tonia-router/skills tonia-sdk`  
  (project scope by default; `--scope user` for user scope)
- Cursor: copy the `tonia-sdk` folder (the directory that contains
  `SKILL.md`) to `.cursor/skills/tonia-sdk/` (project) or
  `~/.cursor/skills/tonia-sdk/` (user). That is an Agent Skill, not a
  Cursor Rule — do not add `tonia-router/skills` as a Remote Rule.
- Manual: same copy, Agent Skills standard (`SKILL.md` at the skill root)

From this repo the folder is `skills/tonia-sdk/`. From the tonia-router
monorepo it is `skills/skills/tonia-sdk/`.

Companion examples: [`sdk-examples`](https://github.com/tonia-router/sdk-examples).
API contract: [`tonia-api`](https://github.com/tonia-router/tonia-api).

## Layout

```text
skills/
├── README.md
└── skills/
    └── tonia-sdk/
        ├── SKILL.md          path picker + safe workflow
        ├── setup.md          portal Members → Issue a key
        ├── tools.md          Cursor / Claude Code / Codex + LLM tools
        ├── typescript.md     @tonia/sdk
        ├── python.md         tonia
        ├── rust.md           not shipped
        ├── errors-and-dlp.md typed errors, streaming, DLP
        └── compatibility.json
```

Copyright (c) 2026 tonia inc.. Apache 2.0 — commercial use allowed. Keep `NOTICE`
if you copy this skill.
