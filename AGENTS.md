When working on or commenting on security, check first file `./agents/security.md`.

When editing `.github/workflows/*.yml`, see `./agents/github-actions-security.md`.

Bash style guide (variables, printf, locals, traps, sourcing, `has`,
shellcheck targets, ...) is hosted canonically in
[`developer-meta-files:agents/bash-style-guide.md`](https://github.com/Kicksecure/developer-meta-files/blob/master/agents/bash-style-guide.md).
We do not duplicate it.

Comprehensive, high-volume regression, fuzz, and end-to-end tests -- too
much for humans to review -- live in the AI-maintained `dist-ai` repo
(https://github.com/org-ai-assisted/dist-ai), not here. Keep only basic
tests in-tree. A package that has such a suite links to it from its own
`AGENTS.md`, together with the environment variable that points the suite
at a local checkout (for example `ONION_GRATER_REPO="$PWD" onion-grater-tests`).

## Tests

This repo's comprehensive suite is hosted canonically in the AI-maintained
[`dist-ai`](https://github.com/org-ai-assisted/dist-ai) repo. We do not
duplicate it. Run it against this checkout with:

    UMOUNT_KILL_SH="$PWD/help-steps/umount_kill.sh" dm-help-steps-tests

Needs root and kills processes -- sandbox VM or throwaway container only.
