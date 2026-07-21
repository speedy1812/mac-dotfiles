---
name: sync-upstream
description: Sync the dotfiles fork with Joshua's upstream repo — preview incoming changes, merge, resolve conflicts using the conflict manifest, verify, and debrief.
disable-model-invocation: true
---

# Sync Upstream

Merge Joshua's upstream dotfiles (`upstream/master`) into this fork (`master`), resolving conflicts according to the manifest below and finishing with a plain-language summary of what changed. The goal is not just a clean merge — it's that Nathan understands what his config now does.

This skill is specific to `~/dotfiles` (fork of `joshukraine/dotfiles`). If run elsewhere, stop and say so.

## Conflict Manifest

How to resolve conflicts, by file. Propose each resolution and wait for approval before staging — never resolve silently.

| Files | Rule |
| --- | --- |
| `claude/.claude/CLAUDE.md`, `claude/.claude/settings.json` | **Keep local.** Personalized for Nathan. Still summarize what upstream changed, in case something is worth adopting manually. |
| `nvim/.config/nvim/lazy-lock.json` | **Take upstream, then regenerate.** After the merge, remind Nathan to open Neovim and let lazy.nvim re-lock; commit the result if it changes. Never resolve this file hunk-by-hunk. |
| `brew/Brewfile` | **Hybrid.** Nathan adds his own packages; Joshua adds/removes his. Merge both sides: keep Nathan's additions, adopt upstream's changes, walk through each hunk. |
| `zsh/.zshrc`, `nvim/.config/nvim/lua/config/options.lua`, `nvim/.config/nvim/lua/plugins/extend-dashboard.lua` | **Hybrid.** Small local tweaks layered on files Joshua actively evolves. Preserve Nathan's edits, adopt upstream's, explain each hunk. |
| Anything else | **Take upstream** — but flag it: a conflict in an uncategorized file means Nathan changed it at some point. Ask which category it belongs in, then update this manifest (see "Manifest maintenance"). |

## Procedure

### 1. Preflight

- Require a clean working tree (`git status`). If dirty, this is a **hard stop**: show Nathan what's uncommitted (including diffs for modified files), recommend a course of action, and wait for his decision. Do **not** commit, stash, or otherwise clean the tree on his behalf — no matter how benign the changes look. A sync should never mix with in-flight work, and what enters history before a sync is Nathan's call.
- Require branch `master`.
- Check `git config rerere.enabled`. If unset, suggest enabling it (records conflict resolutions and replays them on repeat conflicts); enable only with approval.
- `git fetch upstream`.
- If `git log master..upstream/master` is empty, report "already up to date" and stop.

### 2. Preview

Before touching anything, summarize what's incoming:

- Group `git log --oneline master..upstream/master` by top-level package directory (`nvim/`, `zsh/`, `brew/`, `claude/`, …) using `git diff --name-only master...upstream/master`.
- Cross-reference against Nathan's divergence (`git diff --name-only upstream/master...master`) and call out overlaps explicitly: "upstream touched N files you've customized — expect conflicts in: …".
- Highlight anything structural (new packages, deleted files, changes to `setup.sh` or `scripts/`).

Then confirm with Nathan before merging.

### 3. Merge

- `git merge upstream/master`.
- **Clean merge:** skip to step 5.
- **Conflicts:** for each conflicted file, in manifest order (keep-local first, generated next, hybrids last):
  1. Show both sides in plain terms — what upstream changed and why (from commit messages), what Nathan's side has.
  2. Propose a resolution per the manifest rule.
  3. Apply only after approval, then `git add` that file.
- Never run `git checkout --ours/--theirs` on a hybrid file; resolve those hunk-by-hunk.
- If the merge goes sideways, `git merge --abort` restores the pre-sync state — mention this if Nathan seems unsure.

### 4. Complete the merge

Commit with the default merge message (`git commit --no-edit`). Do not squash — merge commits are the record of each sync.

### 5. Verify

- `./scripts/lint-shell`
- `./scripts/run-tests`
- If upstream touched `nvim/`, remind Nathan to open Neovim once to confirm plugins load (and re-lock `lazy-lock.json` if it was conflicted).
- If upstream touched `brew/Brewfile`, offer `brew bundle install`.
- Report failures honestly; don't push with failing tests without asking.

### 6. Debrief

End with a short plain-language summary — this is the "make sense of the changes" step:

- What Joshua changed, grouped by area, in terms of behavior ("his zsh prompt now does X"), not just file names.
- What conflicts were resolved and how.
- Anything Nathan may want to adopt into his keep-local files manually.
- Anything deferred (re-lock nvim, `brew bundle`).

Then offer to push: `git push origin master`.

## Manifest maintenance

The manifest must track reality. After any sync where:

- a conflict appeared in an uncategorized file, or
- Nathan overrode a manifest rule,

update the manifest table in this SKILL.md (with his confirmation) as part of the same session, and include it in a commit. The manifest doubles as documentation of what this fork *is* — the intentional delta from Joshua's repo.
