Goal: Watch the open pull request for the checked-out branch; when new review feedback arrives, implement it, push, and reply — until the PR leaves draft, merges, or closes

## Completed stages
- **check**: succeeded
  - Script: `set -u
if [ -f .fabro-pr-watch ]; then
  read repo pr head < .fabro-pr-watch
else
  sha=$(git rev-parse HEAD)
  origin=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
  line=$(ORIGIN=$origin gh api repos/$origin/commits/$sha/pulls --jq 'map(select(.state=="open"))|(map(select(.base.repo.full_name!=env.ORIGIN))+.)|first|select(.)|[.base.repo.full_name,(.number|tostring),.head.ref]|join(" ")' 2>/dev/null || true)
  if [ -z "$line" ]; then echo "no open PR contains HEAD $sha — will retry"; exit 1; fi
  repo=${line%% *}; rest=${line#* }; pr=${rest%% *}; head=${rest#* }
  echo "$repo $pr $head" > .fabro-pr-watch
fi
state=$(gh pr view $pr --repo $repo --json state,isDraft --jq '[.state,(.isDraft|tostring)]|join(" ")')
if [ "${state%% *}" != "OPEN" ]; then echo "VERDICT: PR $repo#$pr is ${state%% *} — stop watching"; exit 0; fi
if [ "${state#* }" != "true" ]; then echo "VERDICT: PR $repo#$pr left draft — stop watching"; exit 0; fi
own=${repo%%/*}; nam=${repo##*/}
q='query($o:String!,$n:String!,$p:Int!){repository(owner:$o,name:$n){pullRequest(number:$p){reviewThreads(first:100){nodes{isResolved}}}}}'
unresolved=$(gh api graphql -f query="$q" -f o=$own -f n=$nam -F p=$pr --jq '[.data.repository.pullRequest.reviewThreads.nodes[]|select(.isResolved|not)]|length')
headsha=$(gh pr view $pr --repo $repo --json headRefOid --jq .headRefOid)
export TIP=$(gh api repos/$repo/commits/$headsha --jq .commit.committer.date)
newc=$(gh api repos/$repo/issues/$pr/comments --jq '[.[]|select(.created_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
newrev=$(gh api repos/$repo/pulls/$pr/reviews --jq '[.[]|select(.submitted_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
echo "PR $repo#$pr (head branch: $head) draft; unresolved threads=$unresolved, comments since $TIP: $newc, reviews: $newrev"
if [ $((unresolved + newc + newrev)) -gt 0 ]; then echo "VERDICT: feedback to address on $repo#$pr, head branch $head"; exit 0; fi
exit 1`
  - Output:
    ```
    PR fabro-sh/fabro#582 (head branch: feat/resizable-interview-dock) draft; unresolved threads=1, comments since 2026-07-21T07:56:03Z: 0, reviews: 1
    VERDICT: feedback to address on fabro-sh/fabro#582, head branch feat/resizable-interview-dock
    ```
- **pause**: succeeded
- **check**: succeeded
  - Script: `set -u
if [ -f .fabro-pr-watch ]; then
  read repo pr head < .fabro-pr-watch
else
  sha=$(git rev-parse HEAD)
  origin=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
  line=$(ORIGIN=$origin gh api repos/$origin/commits/$sha/pulls --jq 'map(select(.state=="open"))|(map(select(.base.repo.full_name!=env.ORIGIN))+.)|first|select(.)|[.base.repo.full_name,(.number|tostring),.head.ref]|join(" ")' 2>/dev/null || true)
  if [ -z "$line" ]; then echo "no open PR contains HEAD $sha — will retry"; exit 1; fi
  repo=${line%% *}; rest=${line#* }; pr=${rest%% *}; head=${rest#* }
  echo "$repo $pr $head" > .fabro-pr-watch
fi
state=$(gh pr view $pr --repo $repo --json state,isDraft --jq '[.state,(.isDraft|tostring)]|join(" ")')
if [ "${state%% *}" != "OPEN" ]; then echo "VERDICT: PR $repo#$pr is ${state%% *} — stop watching"; exit 0; fi
if [ "${state#* }" != "true" ]; then echo "VERDICT: PR $repo#$pr left draft — stop watching"; exit 0; fi
own=${repo%%/*}; nam=${repo##*/}
q='query($o:String!,$n:String!,$p:Int!){repository(owner:$o,name:$n){pullRequest(number:$p){reviewThreads(first:100){nodes{isResolved}}}}}'
unresolved=$(gh api graphql -f query="$q" -f o=$own -f n=$nam -F p=$pr --jq '[.data.repository.pullRequest.reviewThreads.nodes[]|select(.isResolved|not)]|length')
headsha=$(gh pr view $pr --repo $repo --json headRefOid --jq .headRefOid)
export TIP=$(gh api repos/$repo/commits/$headsha --jq .commit.committer.date)
newc=$(gh api repos/$repo/issues/$pr/comments --jq '[.[]|select(.created_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
newrev=$(gh api repos/$repo/pulls/$pr/reviews --jq '[.[]|select(.submitted_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
echo "PR $repo#$pr (head branch: $head) draft; unresolved threads=$unresolved, comments since $TIP: $newc, reviews: $newrev"
if [ $((unresolved + newc + newrev)) -gt 0 ]; then echo "VERDICT: feedback to address on $repo#$pr, head branch $head"; exit 0; fi
exit 1`
  - Output:
    ```
    PR fabro-sh/fabro#582 (head branch: feat/resizable-interview-dock) draft; unresolved threads=1, comments since 2026-07-21T07:56:03Z: 0, reviews: 1
    VERDICT: feedback to address on fabro-sh/fabro#582, head branch feat/resizable-interview-dock
    ```


# Address PR feedback

The `check` stage output (in your preamble) names the PR, its head branch, and ends with a
`VERDICT:` line.

**If the verdict says the PR is merged, closed, or left draft**: do nothing.
Reply with only this JSON: `{"preferred_next_label": "Done"}`

**Otherwise**, address the feedback. IMPORTANT: the local checkout is fabro's run branch and
contains fabro checkpoint commits — never push it to the PR. Work on a clean branch instead:

1. `git fetch origin <head-branch> && git checkout -B pr-work FETCH_HEAD`
2. Read ALL THREE comment sources — the top-level view alone silently misses line comments:
   - `gh pr view <n> --repo <repo> --comments` (top-level comments)
   - `gh api repos/<repo>/pulls/<n>/comments` (line-level review comments)
   - `gh api repos/<repo>/pulls/<n>/reviews` (review summaries — including requests on APPROVED reviews)
3. For each unresolved thread or comment newer than the remote head commit: implement what it
   asks. Run the checks the change touches (`cd apps/fabro-web && bun test && bun run typecheck`
   for web code; `cargo test -p <crate>` for Rust). Skip nothing silently — if you disagree
   with a comment, say so in the reply instead of implementing it. If the feedback changes
   behavior described in the plan or spec docs in this PR (docs/plans/, docs/brainstorms/),
   update those docs in the same commit; code-level nits never touch them.
4. Commit with a message referencing the feedback. Push: `gh auth setup-git` once, then
   `git push origin pr-work:<head-branch>`.
5. Reply to every thread/comment you acted on, prefixed `[fabro]`, saying what you did (one or
   two sentences). Resolve each thread you addressed:
   `gh api graphql -f query='mutation($t:ID!){resolveReviewThread(input:{threadId:$t}){thread{id}}}' -f t=<thread-id>`
   (thread ids come from the reviewThreads GraphQL query). Do NOT post new top-level comments
   except replies; always prefix your comments `[fabro]` — the watcher filters on it.
6. Return to the run branch: `git checkout -`

When done, reply with only this JSON: `{"preferred_next_label": "Keep watching"}`
