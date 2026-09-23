This is the `build-from-review-base` re-review stage.

Review the commit that repair actually produced, and record a verdict that names
that commit.

## Why this stage exists

`repair-review` is forbidden from grading its own work, and says so: "If your
repair should change that verdict, re-run review against the repaired commit so a
reviewer records it; do not write an approval on review's behalf." Until this
stage existed it was given no mechanism to do that.

Measured on GasCityDispatch workflow gcd-gd1wkd, 2026-09-22: review graded
`68482c7` `changes_required` with four findings. `repair-review` fixed all four,
verified, moved the head to `1ecfdc5`, and set `repair_status=approved`. Nobody
re-graded `1ecfdc5`. The publish gate reads the verdict as well as the repair
status, saw `changes_required`, and refused -- correctly, because the only verdict
on the root described the pre-repair head. The root closed `fail` with good,
verified code sitting in a draft PR.

`max_iterations` is not this mechanism. Both synthesized `review` instances are
consumed at the same moment against the pre-repair head (gcd-387lrf and gcd-jq0g9w
closed four seconds apart, the second with an empty close reason), so no round
remains once the repair exists. A second review that does not depend on the repair
is not a second round.

## Resolve what to grade

```bash
ROOT=<workflow-root-id>
META=$(gc bd show "$ROOT" --json | jq -c '.[0].metadata // .metadata')
PR_URL=$(jq -r '."gc.implementation.pr_url" // ."gc.publish.pr_url"
               // ."gc.build.publish_pr" // ."gc.var.github_pr_url" // ""' <<<"$META")
PUSHED=$([ -n "$PR_URL" ] && gh pr view "$PR_URL" --json headRefOid -q .headRefOid)
REPAIRED=$(jq -r '."gc.build.repair_commit" // ""' <<<"$META")
GRADED=$(jq -r '."gc.build.review_head_sha" // ""' <<<"$META")
REPAIR=$(jq -r '."gc.build.repair_status" // "missing"' <<<"$META")
HEAD=${REPAIRED:-$PUSHED}
```

**The commit to grade is often not the one GitHub is showing.** `publish` pushes
the branch, and `publish` runs after this stage, so a repair that commits leaves
its work on the local branch only and `gh pr view` still reports the pre-repair
head. Measured on GasCityDispatch gcd-p7r3a2 / workflow gcd-4csng2, 2026-09-23:
repair committed `489b402` (5 files, +50, including a new test) at 01:25:53Z and
set `repair_status=approved`, while `refs/pull/322/head` stayed at `5d4e3cb`.

So prefer `gc.build.repair_commit`; fall back to the PR head only when the repairer
recorded no commit. The commit is in the repository's object store even while
unpushed, because the repair ran in a worktree of the same clone. If
`git cat-file -e "$HEAD"` fails, say so and record `changes_required`; do not grade
the pushed head as a substitute and do not approve what you could not read.

**If `$HEAD` is not what the PR is showing, push it before you grade.** The publish
gate dates verdicts against `gh pr view` too, so a verdict naming an unpushed commit
fails its staleness filter, is dropped, and the gate falls back to the unfiltered
set -- where review's original `changes_required` is still standing. An approval of
an unpushed repair refuses exactly as loudly as a rejection.

```bash
if [ -n "$REPAIRED" ] && [ "$REPAIRED" != "$PUSHED" ]; then
  git push origin "$REPAIRED":"$BRANCH"   # never --force; the PR stays draft
fi
```

This is not publishing. The PR stays in draft and no gate is cleared.

## When to skip

**If `REPAIR` is `not_needed` and the verdict already names the current head**,
there is nothing to re-grade. Close `gc.outcome=pass` saying so. This is the normal
path when review approved first time and the repairer had nothing to do.

Note the two conditions. Matching shas alone is NOT enough to skip, because a repair
that only edits the PR body, replies to a thread, or renames a branch makes no
commit and leaves the head exactly where the verdict found it. Measured on
gcd-9hkjy2: review graded `1ecfdc5` `changes_required` on two body-only findings,
and a repair fixing them produces no new sha. Skipping on the sha alone would leave
`changes_required` standing over a PR whose findings are all fixed, and publish
would refuse forever. When `REPAIR` is `approved`, re-grade, whether or not the head
moved.

**If `REPAIR` is `blocked`**, repair already failed honestly and named the findings
that survive. Do not re-grade and do not soften it. Close `gc.outcome=pass` leaving
the existing verdict untouched; the publish gate will refuse on it, which is right.

## Otherwise re-grade

Read the repair's diff against the same standards the first review used -- the
requirements artifact and the repo's own review standard -- and check specifically
that each finding the first review raised is actually fixed, not merely noted. A
test added to satisfy a finding is part of the diff: if it passes with the fix
reverted, it is vacuous and the finding is not repaired.

Then write on the root:

- `gc.build.review_verdict` -- `approved` or `changes_required`, your own reading
- `gc.build.review_head_sha` -- **the sha you just graded**, so the gates can date
  this verdict
- `gc.build.review_report_path` -- where you wrote the report

Both keys together or neither. A verdict without its companion sha is undateable,
and the gates keep undateable verdicts unconditionally, so half of this record is
worse than none.

**Do not write `approved` because `repair_status` says `approved`.** The repairer
grading its own work is the failure every gate in this formula exists to prevent.
If the repair did not fix a blocking finding, say `changes_required` and name it.

## Close pass either way

Close `gc.outcome=pass` whichever verdict you record. The verdict is the signal; the
step outcome is not. A `fail` here becomes a permanent blocking edge on the item root
that no later step can clear. Record what you found and let the publish gate refuse
on it.
