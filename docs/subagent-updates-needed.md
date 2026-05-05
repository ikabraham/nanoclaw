# Subagent Refactoring — Updates Needed

Tracks what was done in the recent subagent identity/channel refactoring and what work remains.

---

## What Was Done

### Telegram channel added
New files in `src/channels/`:
- `telegram.ts` — full Chat SDK adapter with polling, reply context extraction, and exponential-backoff cold-start retry
- `telegram-pairing.ts` — one-time 4-digit code pairing system; proves operator owns the chat before registration; stores state at `data/telegram-pairings.json`
- `telegram-markdown-sanitize.ts` — strips Markdown that Telegram's legacy parse_mode rejects
- Tests for both pairing and sanitize modules
- `@chat-adapter/telegram@4.26.0` added to `package.json`
- `src/channels/index.ts` now imports `./telegram.js`

### CLAUDE.local.md compose order changed
`src/claude-md-compose.ts`: `CLAUDE.local.md` now loads **before** `.claude-shared.md` in the composed `CLAUDE.md`. Per-group role and identity now take precedence over shared capabilities — agents see their own constraints before the general toolkit.

### container/CLAUDE.md updated
Description of `CLAUDE.local.md` updated to frame it as **role, identity, and behavioral constraints** (not just memory). Agents are now explicitly told to treat it as authoritative and binding.

### ComfyRe identity file refactored
`groups/comfyre/CLAUDE.local.md` reduced from 728 lines to ~35 lines. The new file states identity, pipeline position, template routing table, dispatch protocol summary, and report format. The detailed workflow JSON, extension injection rules, validation checks, and credential vault paths were removed from the identity file.

### Global group cleanup
`groups/global/CLAUDE.md` deleted. The global group is removed.

---

## Pending Work

### 1. ComfyRe workflow templates — move to workspace memory

**Priority: high**

The six ComfyUI node-graph templates (SD1.5 text2img, SDXL text2img, img2img, Wan 2.1 t2v, Wan 2.2 t2v, Wan 2.1 i2v), the extension injection rules (LoRA, ControlNet, clip-skip), validation rules, dependency list, and vault credential paths were stripped from `CLAUDE.local.md` but not migrated anywhere. ComfyRe cannot build workflows without them.

**Fix:** Create `groups/comfyre/memory/comfyre-workflow-reference.md` with the full template set. ComfyRe can read this file from `/workspace/group/memory/comfyre-workflow-reference.md` at the start of a generation session.

The reference content to restore is in git history: `git show HEAD:groups/comfyre/CLAUDE.local.md` shows the full prior content before compression (note: that was the pre-refactor file in the working tree at the time of the first commit; the git diff above shows the exact removed content).

### 2. Remaining subagent CLAUDE.local.md files — apply new identity format

**Priority: medium**

The new standard (tight identity, role constraints first, behavioral bindings before capabilities) was only applied to ComfyRe. The following groups still have old-style long-form descriptions that predate the compose-order change and the new framing:

| Group | File | Notes |
|-------|------|-------|
| `creyah` | `groups/creyah/CLAUDE.local.md` | Old verbose header; doesn't use the new "This is your identity" framing |
| `creativere` | `groups/creativere/CLAUDE.local.md` | Needs review |
| `fetchre` | `groups/fetchre/CLAUDE.local.md` | Needs review |
| `copy` | `groups/copy/CLAUDE.local.md` | Needs review |
| `funnel` | `groups/funnel/CLAUDE.local.md` | Needs review |
| `tiktok` | `groups/tiktok/CLAUDE.local.md` | Needs review |
| `cro` | `groups/cro/CLAUDE.local.md` | Needs review |
| `devops` | `groups/devops/CLAUDE.local.md` | Needs review |

Apply the same pattern: tight identity paragraph up front, explicit "do not act as a general assistant" gate, then role-specific reference content.

### 3. Main group CLAUDE.local.md — v1 content cleanup

**Priority: medium**

`groups/main/CLAUDE.local.md` contains extensive v1-era content: references to `registered_groups.json`, `available_groups.json`, the WhatsApp IPC task-drop pattern, and the old sender-allowlist config at `~/.config/nanoclaw/`. None of these paths exist in v2. The v2 entity model uses the central SQLite DB (`data/v2.db`) and the router handles group wiring — the agent does not manage `registered_groups.json` directly.

**Fix:** Audit `groups/main/CLAUDE.local.md` against the v2 architecture (see `docs/api-details.md` and `docs/db-central.md`) and remove or replace all v1-specific instructions.

### 4. Build and test the Telegram channel

**Priority: high before first Telegram session**

```bash
pnpm run build
pnpm test
```

Verify:
- `src/channels/telegram.ts` compiles cleanly
- `telegram-pairing.test.ts` and `telegram-markdown-sanitize.test.ts` pass
- No regressions in existing channel tests

### 5. Telegram add-telegram skill — verify pairing integration

**Priority: medium**

The `/add-telegram` skill (on the `channels` branch) predates the pairing system. Verify it references `createPairing` / `waitForPairing` from `telegram-pairing.ts` and walks the operator through the pairing code flow. If it doesn't, update the skill to use the pairing API instead of any manual wiring step.

### 6. OrchaYah group — confirm existence or create

**Priority: low**

ComfyRe and CreReYah both reference OrchaYah as their coordinator, but `groups/` does not contain an `orchayah` folder. Confirm whether OrchaYah is wired to an existing group under a different folder name, or create the group with a proper CLAUDE.local.md defining its orchestration role.

---

## Reference

- Telegram pairing API: `src/channels/telegram-pairing.ts` — `createPairing`, `waitForPairing`, `tryConsume`
- Compose logic: `src/claude-md-compose.ts:119` — CLAUDE.local.md import order
- ComfyRe identity: `groups/comfyre/CLAUDE.local.md`
- Full workflow templates (git history): `git show 8cb42b2:groups/comfyre/CLAUDE.local.md`
