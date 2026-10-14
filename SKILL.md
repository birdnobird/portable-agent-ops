---
name: portable-agent-ops
description: "Operate a portable agent setup — a cross-device Markdown memory repository plus the skills installed or published from a machine. Use when starting or ending a work session, when asked to remember, forget, search, or update personal context, when syncing or health-checking the memory system, and when creating, installing, sanitizing, or publishing a skill to GitHub. Also use when a repository's visibility must be verified before a push, or when a private repository is being turned into a public one."
version: 1.0.0
user-invocable: true
license: Apache-2.0
---

# Portable agent ops

Two related jobs: keeping **memory** portable across machines and agents, and keeping **skills** portable by publishing them. Both exist so that switching computers does not mean starting over.

## Configuration

Fill this block in for the machine you are on. Everything below refers to these names.

| Name | Value |
|---|---|
| Memory repository | `https://github.com/<owner>/<repo>` (must be PRIVATE) |
| Local clone | `<absolute path to the local clone>` |
| This device id | `<stable per-device id, e.g. os-agent-xxxx>` |
| GitHub account | `<owner>` |

Mount this skill through the agent's own skill directory so it is available in any working directory, not just inside the repository.

## Part A — The memory repository

Plain Markdown, no vendor lock-in, so any agent on any machine can read it. A layout that works:

| Path | Purpose |
|---|---|
| `USER.md` | Stable preferences: how to address the user, communication style, long-term goals, explicit do-not-record list |
| `MEMORY.md` | Index of topic files, one line each |
| `PROJECTS.md` | Active / paused / completed work, with decisions and blockers |
| `memory/inbox/<device>/` | Append-only notes, **one directory per device** |
| `memory/topics/` | Consolidated, durable subject files |
| `tools/scan-secrets.py` | The scanner referenced below |

### Non-negotiable safety rules

1. Before **any** push carrying personal context, verify with an authenticated official interface that repository visibility is exactly `private`. If it is public, internal, unknown, or unverifiable — **stop**.
2. Never request, record, or echo passwords, one-time codes, tokens, API keys, private keys, cookies, identity documents, financial data, or unrelated third-party private information. If unsure whether something is sensitive, ask rather than decide.
3. Never use force push, hard reset, destructive checkout, or branch deletion to resolve a sync problem.
4. Never overwrite or delete files without showing the effect and getting approval.
5. If a secret may have entered history, deleting the file is not enough — say so plainly, tell the user to rotate it, then clean history by the official procedure.
6. When scanning for secrets, report only whether something was found and where. **Never print the suspicious value** — that turns a private scan into a public disclosure.

### Sync procedure

Run from the repository root.

```bash
git status --porcelain              # see what is actually pending
git pull --no-rebase                # integrate; keep both sides on conflict
python tools/scan-secrets.py --staged
gh api repos/<owner>/<repo> --jq '.visibility'   # must print: private
git commit -F <message-file>
git push origin main
git rev-parse HEAD && git rev-parse origin/main  # must match
```

Scanner exit codes: `0` clean, `1` hits found, `2` tool failure.

**Pass commit messages via a file (`-F`), not `-m`.** Multi-line messages containing quotes, asterisks, or non-ASCII get split by the shell into pathspecs, and the commit silently fails with `error: pathspec '...' did not match any file(s)` while the subsequent push reports "Everything up-to-date".

### Conflict policy

Never silently pick a side. Keep both versions, explain the disagreement in plain language, and ask which is current. Prefer new timestamped inbox files over two devices editing one journal.

When `git` reports a conflict, **exit 1 with `<<<<<<<` markers and both texts intact is the expected, desired outcome** — not a failure. Resolve by hand, keeping both sides, then push.

A rehearsal of this (two devices, one appending to its own inbox and one editing a shared file) is worth running whenever the layout changes. Keep the rehearsal script rather than trusting memory — and run it against a throwaway clone, never the real repository.

### Network and credential facts worth re-checking per machine

- Some Windows setups need `git config --global http.sslbackend openssl`. The SChannel default can fail against GitHub with `SEC_E_NO_CREDENTIALS`. **Verify on the target machine rather than assuming.**
- Prefer a credential helper already authorized on the machine (for example `gh auth setup-git`) over asking the user for a token. **The user should never paste a token into chat.**
- A push can fail with `Connection was reset`; retry a few times before investigating.
- An error mentioning `couldn't create signal pipe` or a failed `prompt script` does **not** by itself mean the push failed. Check the actual refs.
- Writing outside the agent's sandbox (persistent environment variables, files in another tool's config directory) may require elevated file permissions.

## Part B — Skills

### Frontmatter rules

```yaml
---
name: portable-agent-ops
description: "…always quote this…"
version: 1.0.0
user-invocable: true
license: Apache-2.0
---
```

- **Quote `description`.** Unquoted, a `": "` inside it is parsed as a YAML mapping separator, the frontmatter fails to parse, and the skill vanishes. The symptom is `skill "X" is unknown or no longer available` — **with no syntax error and no log line**. Expect to lose hours to this if you do not know it.
- Model invocation and user invocation both default to **true**. Adding `user-invocable: true` is harmless but redundant. Do not invent `userInvocable` / `modelInvocable` — those spellings are explicitly rejected by strict parsers.
- Only the `description` reaches the model's visible catalog. Whether a skill triggers correctly is entirely a description problem, so write it as "use when…" including the negative cases ("not for casual chat").
- A skill listed in a UI is not necessarily visible to the model; those can be two independent filters.

### Where skills live

| Location | Visible to the agent | Notes |
|---|---|---|
| The user's agent skill directory (`~/.<agent>/skills/<name>/SKILL.md`) | Usually | Requires an active filesystem skill provider; a `watch` option may make new folders live without a restart |
| A plugin bundle's `skills/` directory | Yes | Follows the plugin's enable/disable state |
| A project-local skill folder | Not by default | Needs an explicit custom skill directory in the provider config |

If a skill is silently not found, check whether any **active** provider actually scans that directory. "The directory is not being watched" and "you must restart" are both common wrong guesses.

### Publishing a skill publicly

Order matters. Do the sanitizing **before** anything is reachable by the network.

1. **Work on a copy.** Duplicate the source into a scratch directory. Leave the installed skill byte-identical, and hash-verify that at the end.
2. **Strip machine-specific content.** Local paths, usernames, device ids, private repository URLs, personal identifiers. A skill that names the user's private repository **must not** be published as-is.
3. **Quote the description** (above).
4. **Fetch the license, do not transcribe it.** Pull the canonical text from the license's own URL and validate it — expected length plus several required phrases — before writing. Hand-copying a license invites silent truncation and corruption.
5. **Add `.gitattributes`** containing `* text=auto eol=lf`, or every `git add` emits CRLF warnings.
6. **Repository names must be ASCII** — letters, digits, `-`, `_`, `.` only; most hosts reject non-ASCII. A non-English name belongs in the description and README title instead, not in the repository name.
7. Create and push:
   ```bash
   gh repo create <owner>/<name> --public --description "<description>"
   git remote add origin https://github.com/<owner>/<name>.git
   git push -u origin main
   ```
8. **Verify twice — with credentials and without:**
   - authenticated: `gh api repos/<owner>/<name> --jq '.visibility'` → `public`
   - **anonymous**, sending no `Authorization` header at all: a public repository returns `200`, a private one returns `404`. That difference is the real proof, because the authenticated call only reports what you configured.

Publishing is effectively irreversible — a public repository can be cloned, cached, and mirrored within minutes. Confirm before the create step, not after.

### Verifying a published file

Fetch the raw file and compare hashes against the local copy. **Collect buffers; never concatenate response chunks as strings.** A string round-trip splits multi-byte UTF-8 characters, producing `EF BF BD` replacement bytes and a larger file — which looks exactly like the repository being corrupt when it is not.

```js
const chunks = [];
res.on('data', (c) => chunks.push(c));      // keep the Buffer
res.on('end', () => fs.writeFileSync(out, Buffer.concat(chunks)));
```

Raw-content hosts intermittently reset connections; retry a few times before concluding anything.

**When a hash mismatch appears, suspect your download path before you suspect the repository.**

## Before finishing

- [ ] Repository visibility verified with an authenticated call, and anonymously where publication is claimed.
- [ ] Any published file re-downloaded and hash-compared against the local source.
- [ ] The installed copy of any skill is unchanged (hash-verified) when a publish copy was made.
- [ ] No secret values were printed, only their locations.
- [ ] Commit messages passed by file, not `-m`.
- [ ] One-off probe scripts deleted; reusable ones kept.
