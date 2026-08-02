# skills

Portable Agent Skills for tonia. The official skill is **`tonia-sdk`**.

## Install

- Portable: `gh skill install tonia-router/skills tonia-sdk`  
  (project scope by default; `--scope user` for user scope)
- Cursor: **Settings → Rules → Add Rule → Remote Rule (GitHub)** with
  `tonia-router/skills`
- Manual: copy `skills/tonia-sdk/` following the Agent Skills standard

Companion examples: [`sdk-examples`](https://github.com/tonia-router/sdk-examples).
API contract: [`tonia-api`](https://github.com/tonia-router/tonia-api).

## Layout

```text
skills/
├── README.md
└── skills/
    └── tonia-sdk/
        ├── SKILL.md
        ├── typescript.md
        ├── python.md
        ├── rust.md
        └── errors-and-dlp.md
```
