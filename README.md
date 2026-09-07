# claude-code-review

Automated code review powered by Claude, running on a Claude Max plan. No
`ANTHROPIC_API_KEY` is used anywhere: the local hook uses the `claude` CLI login,
and CI uses an OAuth token stored as a repository secret.

## Code review workflow

Every review, wherever it runs, reads the same prompt from
[`.claude/review-prompt.md`](.claude/review-prompt.md). Edit that one file to
change how reviews behave. The reviewer ends its output with exactly one of
`MERGE`, `MERGE AFTER FIXES`, or `DO NOT MERGE`.

There are three ways a review happens.

**Manual, on demand.** Run `/review-jt` in Claude Code to review the current
branch without pushing. Use it while you are still working.

**On push.** The hook at [`.githooks/pre-push`](.githooks/pre-push) diffs
`origin/main...HEAD`, sends it to Claude, prints the review, and saves a copy
under `.reviews/` (git-ignored). A `DO NOT MERGE` verdict blocks the push;
`MERGE AFTER FIXES` does not. The hook is shared through the repo rather than
living in `.git/hooks`, so each fresh clone needs this once:

```bash
git config core.hooksPath .githooks
```

**On pull request.** The workflow at
[`.github/workflows/claude-review.yml`](.github/workflows/claude-review.yml)
reviews every non-draft PR and posts one review with inline annotations on the
changed lines. Dependabot PRs are skipped.

### Escape hatch

To push without a review, for a hotfix or when the reviewer is wrong:

```bash
SKIP_REVIEW=1 git push
```

The hook also blocks the push if `claude` fails outright, rather than passing
silently on an infrastructure error. Use the same escape hatch if that happens.

### Caveats

- Only committed work is reviewed. Uncommitted working-tree edits are invisible to `git diff base...HEAD`.
- The hook needs `origin/main` to exist. Without a remote it exits 0 and reviews nothing.
- Pushing from inside a Claude Code session can fail, because the hook starts a nested `claude -p`. Push from a plain terminal.
