# portable-agent-ops

**A skill for keeping an AI agent's memory and skills portable across machines.**

Two problems this solves, both of which only show up once you use more than one computer:

1. **Your agent forgets you.** Preferences, decisions, project state — all trapped on one machine, in one vendor's format.
2. **Your skills are trapped too.** You write a skill, then can't hand it to anyone or carry it to the next machine.

This skill encodes one working answer to both, including the specific traps that cost real debugging time.

## What's in it

**Part A — a memory repository.** Plain Markdown, no vendor lock-in, synced through a **private** Git repository. Files by role (`USER.md`, `MEMORY.md`, `PROJECTS.md`), append-only per-device inbox directories so two machines never fight over one file, and a secret scanner that runs before every commit.

**Part B — skills.** Frontmatter rules that actually matter, where a skill must live to be visible to the model, and a step-by-step procedure for sanitizing and publishing one publicly.

## The traps it records

These are the reason this exists as a skill rather than as advice. Every one of them produced a wrong diagnosis first.

| Trap | Symptom | Real cause |
|---|---|---|
| Unquoted `description` in frontmatter | `skill "X" is unknown or no longer available` — no syntax error, no log | A `": "` inside an unquoted YAML scalar is read as a mapping separator; the whole frontmatter is dropped |
| Multi-line commit message via `-m` | `error: pathspec '...' did not match any file(s)`, then a push reporting "Everything up-to-date" | The shell split the message into arguments |
| Non-ASCII repository name | GitHub rejects creation | Repository names are limited to an ASCII subset; the display name belongs in the description |
| String-concatenated HTTP chunks | Hash mismatch when verifying a downloaded file — looks like the repo is corrupt | Multi-byte UTF-8 characters were split across chunks and replaced with `EF BF BD` |
| `curl` reporting the network is down | Every `curl` call fails while another runtime succeeds | Different TLS stacks; the probe was wrong, not the network |
| A push producing no output for minutes | Nothing happens; no error, no progress | The connection silently stalls — set `http.lowSpeedLimit` / `http.lowSpeedTime` so it fails fast and can be retried |

The last two share a lesson the skill states outright: **when a hash mismatch appears, suspect your download path before you suspect the repository.**

## Safety rules it enforces

- Verify a repository is `private` with an authenticated call **before** any push carrying personal context. If unverifiable, stop.
- Never print a suspected secret — report only the file and line. Printing it turns a private scan into a public disclosure.
- Never resolve a sync conflict by choosing a side silently, and never use force push or hard reset to make one go away.
- A published repository is effectively irreversible. Confirm before creating it, not after.

## Install

Drop `SKILL.md` into your agent's skill directory:

```bash
git clone https://github.com/<owner>/portable-agent-ops.git \
  ~/.claude/skills/portable-agent-ops
```

Same file works for any agent that reads Markdown skills (Claude Code, DSH, and similar). Paths differ per implementation.

Then **fill in the Configuration table** at the top with your own repository URL, local path, and a device id. The version here uses placeholders.

> ⚠️ **Quote the `description` field.** If you edit it, keep the quotes. See the traps table above — without them the skill silently stops loading.

## Scope

Use it for: session start/end, remembering or forgetting something, syncing, health-checking, and creating, installing, sanitizing, or publishing a skill.

Not for: casual conversation, or any turn with no memory or packaging decision in it.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
