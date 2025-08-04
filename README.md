# Git Tricks

A collection of useful Git tricks

## Diff Two Commit Ranges with Different Base Branches

If two Git branches:

1. share some cherry-picked commits but also differ
2. are based on different base branches

...it can be useful to generate a diff as if they were to land on the same target (eg. `main`). Using synthetic trees with [`git merge-tree`](https://git-scm.com/docs/git-merge-tree) (Git >= 2.38), shared patches cancel out, so the output highlights only what's unique to each range:

```bash
# compare_changesets: compare two change-sets on a common target (Git >= 2.38)
# ----------------------------------------------------------------------------
# API:
#   compare_changesets <TARGET> <BASE_A> <TIP_A> <BASE_B> <TIP_B>
#
# Example:
#   compare_changesets main production production-login-ui main login-ui
#
# Diagram & explanation:
#
#                           C1'    C2'    D1    D2
# production-login-ui        o------o------o-----o
#                           /
# production         o--o--o
#                   /
# main      - - ---o
#                   \
# login-ui           o------o
#                   C1     C2
#
# - Shared patches: C1/C2 on login-ui and C1'/C2' on production-login-ui represent
#   the same logical changes (eg. cherry-picked or equivalent patches). When we
#   synthesize both change-sets onto TARGET (main), those shared patches produce the
#   same resulting content on both synthetic trees and therefore cancel out of the
#   comparison.
#
# - What the diff shows: with A = (production → production-login-ui) and
#   B = (main → login-ui) compared on TARGET = main, the resulting `git diff -M`
#   between the two synthetic trees highlights only what is unique between the
#   change-sets on that common base. In this scenario, that is D1 and D2 — eg.
#   the additional changes after C2' on production-login-ui (modulo any conflict
#   resolutions that could introduce minor context differences).
compare_changesets() {
  set -o nounset

  if [ "$#" -ne 5 ]; then
    echo "Usage: compare_changesets <TARGET> <BASE_A> <TIP_A> <BASE_B> <TIP_B>" >&2
    return 2
  fi

  local target="$1" base_a="$2" tip_a="$3" base_b="$4" tip_b="$5"

  # Build synthetic trees by merging each change-set onto target with its own base
  local tree_a
  tree_a="$(git merge-tree --write-tree --merge-base "$base_a" "$target" "$tip_a")"

  local tree_b
  tree_b="$(git merge-tree --write-tree --merge-base "$base_b" "$target" "$tip_b")"

  # Show the net difference between the two effects on target
  git diff -M "$tree_a" "$tree_b"
}
```
