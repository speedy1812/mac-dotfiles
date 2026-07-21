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
| `brew/Brewfile` | **Hybrid.** Nathan adds his own packages; Joshua adds/removes his. Merge both sides: keep Nathan's additions, adopt upstream's changes, walk through each hunk. New upstream entries get per-item review even when they merge cleanly — see step 4 "Drift check". |
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
- **Clean merge:** still do step 4 (keep-local drift check) — clean merges are exactly where drift hides — then skip to step 6.
- **Conflicts:** for each conflicted file, in manifest order (keep-local first, generated next, hybrids last):
  1. Show both sides in plain terms — what upstream changed and why (from commit messages), what Nathan's side has.
  2. Propose a resolution per the manifest rule.
  3. Apply only after approval, then `git add` that file.
- Never run `git checkout --ours/--theirs` on a hybrid file; resolve those hunk-by-hunk.
- If the merge goes sideways, `git merge --abort` restores the pre-sync state — mention this if Nathan seems unsure.

### 4. Drift check

A clean auto-merge is not the same as "nothing to review." When Nathan's edits and upstream's touch *different lines* of a file, git merges them silently and the manifest never fires. Both halves of this check run before the merge commit, conflicted or not — clean merges are exactly where drift hides.

**Keep-local files.** Upstream's personal content can flow into personalized files unnoticed (this happened: Joshua's "Language" section about *his* Ukrainian fluency auto-merged into Nathan's CLAUDE.md). For every keep-local file in the manifest, run `git diff HEAD -- <file>`:

- Nothing changed → fine, move on.
- Upstream content arrived → summarize it and ask Nathan per item: keep it, drop it, or personalize it (e.g., rewrite a Joshua-specific section for Nathan). Apply his choices and stage the file before the merge commit.

**Brewfile additions.** New active entries from Joshua auto-merge in silently and become software installed on Nathan's machine at the next `brew bundle install` (this happened: `1password-cli` slid in unnoticed; Nathan doesn't use 1Password). Run `git diff HEAD -- brew/Brewfile` and for each **newly added active line** (`brew`/`cask`/`mas`/`tap`, not comments), tell Nathan:

1. What was added.
2. What the software does, in a sentence or two.
3. A recommendation — install (e.g., it supports something else arriving in this sync, or fits how Nathan works) or skip — with the reasoning.

Then ask per item: **keep active** (Nathan wants it) or **comment it out** (Joshua-only). Commenting out is Nathan's established convention for "Joshua uses this, I don't" — the line stays as documentation and merges cleanly later. Removals and edits to existing entries just get summarized; they don't install anything. Apply choices and stage before the merge commit.

### 5. Complete the merge

Commit with the default merge message (`git commit --no-edit`). Do not squash — merge commits are the record of each sync.

### 6. Verify

- `./scripts/lint-shell`
- `./scripts/run-tests`
- If `lazy-lock.json` changed in the merge, have Nathan run `nvim --headless "+Lazy! restore" +qa` (or `:Lazy restore` inside Neovim) so installed plugin versions match the new pins — merely opening Neovim does nothing. If the lockfile differs afterward (`git status`), commit it as `chore(nvim): re-lock plugins after upstream sync`.
- If upstream touched `brew/Brewfile`, offer `brew bundle install`.
- If upstream deleted a package directory, check `$HOME` for dangling stow symlinks to it and offer to remove them.
- Report failures honestly; don't push with failing tests without asking.

### 7. Debrief

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
