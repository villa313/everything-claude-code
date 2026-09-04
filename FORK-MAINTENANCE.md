# Fork Maintenance

How this fork is kept current with upstream ECC and rolled out to the local Claude
environments. Written for whoever (human or agent) picks this up next.

> **This file is fork-only.** It does not exist upstream, so it never conflicts on merge.
> Same reasoning applies to anything else you add here — prefer new files over edits to
> upstream-tracked ones.

---

## 1. What this repo is

A fork of **[affaan-m/ECC](https://github.com/affaan-m/ECC)** carrying a custom **C# language
track** that must survive every upstream merge.

| Remote | Points at |
|---|---|
| `origin` | `villa313/everything-claude-code` (the fork — what the local envs install from) |
| `upstream` | `affaan-m/ECC` (source of truth for everything else) |

### The C# track — protect these

```
agents/csharp-build-resolver.md          commands/csharp-build.md
agents/csharp-reviewer.md                commands/csharp-review.md
rules/csharp/{coding-style,hooks,        commands/csharp-test.md
  patterns,security,testing}.md          skills/csharp-testing/SKILL.md
.kiro/agents/csharp-{build-resolver,reviewer}.{md,json}
```

Plus entries in `agent.yaml` and `docs/COMMAND-REGISTRY.json`. Localized mirrors under
`docs/{ja-JP,zh-CN}/` bring the total to **29 files matching `csharp`** — a useful smoke check.

`csharp-reviewer.md` also exists upstream, so a careless merge can silently take their
version. Always diff it against the pre-merge state, not just check that the file exists.

---

## 2. How updates reach the machine

```
upstream/main ──merge──> fork main ──push──> origin
                                               │
                        ┌──────────────────────┴──────────────────────┐
                   ~/.claude-per                              ~/.claude-accrueme
                    (personal)                                     (work)
```

Both environments install ECC as the plugin **`ecc@everything-claude-code`** (scope `user`)
from a marketplace pointing at **the fork**, not upstream. Nothing reaches them until you push.

- Select an environment with `CLAUDE_CONFIG_DIR=~/.claude-per claude ...`
- The directory is `.claude-accrueme` — **not** `.claude-acc`
- `~/.claude` is a separate legacy *file-copy* install that neither environment uses
- Two layers exist per env: `plugins/marketplaces/<name>` (a git clone) and
  `plugins/cache/<marketplace>/<plugin>/<version>` (the content actually loaded).
  `marketplace update` refreshes the first; `plugin update` refreshes the second. Both needed.

---

## 3. The tool

**`~/.local/bin/ecc-sync`** — deliberately outside the repo, so it never becomes a permanent
diff that has to survive merges.

```
ecc-sync           fetch, merge, auto-resolve counts, test, STOP before push
ecc-sync --status  what is staged and waiting
ecc-sync --push    approve: push to origin, then update both envs
ecc-sync --abort   discard a staged sync, restore the pre-sync state
ecc-sync --envs    update the two environments only
```

State lives in `~/.local/state/ecc-sync/`: `pending.json` (staged sync),
`known-failures.txt` (test baseline), `logs/`.

Safety properties worth preserving if you rewrite it:

- Stashes local uncommitted changes across the merge and restores them after
- Creates a `backup/pre-sync-<date>` branch before merging
- Aborts and changes nothing if conflicts fall outside the known count/version allowlist
- Never pushes on its own — the push is always a human decision
- Refuses to compare test results when `node_modules` is missing, and flags >50 failures as
  a broken environment rather than a bad merge

### Scheduling

A LaunchAgent at `~/Library/LaunchAgents/com.luis.ecc-sync.plist` runs
`/bin/bash -lc ~/.local/bin/ecc-sync` weekly (**Mondays 10:00**, `StartCalendarInterval`).
`RunAtLoad` is false. Output appends to `~/.local/state/ecc-sync/logs/launchd.{out,err}.log`.
If the Mac is asleep, launchd runs it at next wake instead of skipping the week.

```bash
launchctl start com.luis.ecc-sync    # force a run
launchctl list  com.luis.ecc-sync    # state + LastExitStatus
launchctl unload ~/Library/LaunchAgents/com.luis.ecc-sync.plist   # disable
launchctl load   ~/Library/LaunchAgents/com.luis.ecc-sync.plist   # re-enable
```

Schedule edits require `unload` + `load`; editing the plist alone does nothing.

---

## 4. Gotchas — each of these cost real debugging time

**Merge conflicts are always catalog-count churn.** Manifests, `AGENTS.md`, `README.md`, and
locale copies. Resolve by taking upstream's text, then re-deriving counts from the tree.
Never copy either side's numbers.

**Fixing only the conflicted files is not enough.** Other docs state the same counts *without
conflicting*, so git keeps upstream's number while the tree has one more agent (the C#
`csharp-build-resolver`). That fails `tests/ci/validators.test.js`. Always run:

```bash
node scripts/ci/catalog.js --write --text   # rewrites the 7 files it tracks
node scripts/ci/catalog.js                  # JSON report, no writes
```

**`docs/tr/AGENTS.md` is not tracked by that script** but *is* asserted by
`tests/docs/configure-ecc-install-paths.test.js`. Patch it by hand.
**Leave `docs/es` and `docs/ja-JP` alone** — their counts are intentionally stale upstream;
editing them creates a conflict on every future merge.

**The test suite rewrites `yarn.lock`** as a side effect (reformats it, ~3.6k lines). Always
`git checkout -- yarn.lock` before committing, or it silently rides along in a `git add -A`.

**Run test suites one at a time.** Two concurrent runs in the same working directory produced
149 phantom failures; the same tree run alone gave 4.

**PATH under launchd.** The global npm `claude` at `/usr/local/bin` was removed (it had drifted
to 2.1.68 while the native install was 2.1.260). Bare `claude` is now *not found* in a minimal
launchd environment, so `ecc-sync` prepends `$HOME/.local/bin` to PATH. Keep that line.

### Known-failing tests — pre-existing upstream, not caused by any merge

Four in `tests/lib/state-store.test.js`: three `status CLI ...` cases and `work-items CLI syncs
GitHub PRs and issues into readiness`. They assert "2 attention items" and get 3. Verified by
running that file against a pristine `upstream/main` worktree. Baseline:
`~/.local/state/ecc-sync/known-failures.txt`.

---

## 5. Doing a merge by hand

If `ecc-sync` bails on an unexpected conflict:

```bash
git fetch upstream --prune
git branch backup/pre-sync-$(date +%F)         # always
git stash push -u                              # local CLAUDE.md / .claude/settings.json
git merge upstream/main --no-edit

# resolve; for count/version files take upstream, then:
node scripts/ci/catalog.js --write --text
#   ...and hand-fix docs/tr/AGENTS.md

git diff --stat <pre-merge-sha> HEAD -- 'agents/csharp*' 'commands/csharp*' 'rules/csharp/*'
#   ^ MUST be empty: the C# track should be byte-identical

node tests/run-all.js          # ~7 min; compare failures against the baseline above
git checkout -- yarn.lock      # the suite dirtied it
git stash pop
```

Verify before pushing:

```bash
git rev-list --count main..upstream/main            # 0 = fully synced
git ls-tree -r --name-only HEAD | grep -ci csharp   # expect 29
```

Roll back a bad merge with `git reset --hard backup/pre-sync-<date>`.

---

## 6. Verifying the whole chain

```bash
# fork state
git rev-list --count main..upstream/main    # 0
git rev-parse main origin/main              # identical

# both environments
for d in ~/.claude-per ~/.claude-accrueme; do
  CLAUDE_CONFIG_DIR=$d claude plugin list | grep -A1 'ecc@'
done

# schedule
launchctl list com.luis.ecc-sync | grep LastExitStatus   # 0
```

Restart any running Claude session after a plugin update — the new version is not hot-loaded.

---

## 7. Useful references

- Upstream repo: <https://github.com/affaan-m/ECC>
- This fork: <https://github.com/villa313/everything-claude-code>
- `scripts/ci/catalog.js` — the count validator; source of truth for agent/skill/command totals
- `tests/run-all.js` — full suite; individual files run directly with `node tests/<path>`
- Claude Code plugin CLI: `claude plugin --help`, `claude plugin marketplace --help`
- launchd scheduling: `man launchd.plist` (see `StartCalendarInterval`)
