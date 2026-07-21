Goal: Watch the open pull request for the checked-out branch; when new review feedback arrives, implement it, push, and reply — until the PR leaves draft, merges, or closes

## Completed stages
- **check**: succeeded
  - Script: `set -u
branch=$(git rev-parse --abbrev-ref HEAD)
origin=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
parent=$(gh api repos/$origin --jq '.parent.full_name // empty')
pr=''; repo=''
for r in $parent $origin; do
  n=$(gh pr list --repo $r --search head:$branch --state open --json number --jq '.[0].number // empty')
  if [ -n "$n" ]; then pr=$n; repo=$r; break; fi
done
if [ -z "$pr" ]; then echo "VERDICT: no open PR for $branch (merged/closed?) — stop watching"; exit 0; fi
draft=$(gh pr view $pr --repo $repo --json isDraft --jq .isDraft)
if [ "$draft" != "true" ]; then echo "VERDICT: PR $repo#$pr left draft — stop watching"; exit 0; fi
own=${repo%%/*}; nam=${repo##*/}
q='query($o:String!,$n:String!,$p:Int!){repository(owner:$o,name:$n){pullRequest(number:$p){reviewThreads(first:100){nodes{isResolved}}}}}'
unresolved=$(gh api graphql -f query="$q" -f o=$own -f n=$nam -F p=$pr --jq '[.data.repository.pullRequest.reviewThreads.nodes[]|select(.isResolved|not)]|length')
export TIP=$(git log -1 --format=%cI)
newc=$(gh api repos/$repo/issues/$pr/comments --jq '[.[]|select(.created_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
newrev=$(gh api repos/$repo/pulls/$pr/reviews --jq '[.[]|select(.submitted_at > env.TIP)|select(.body|contains("[fabro]")|not)]|length')
echo "PR $repo#$pr draft; unresolved threads=$unresolved, comments since $TIP: $newc, reviews: $newrev"
if [ $((unresolved + newc + newrev)) -gt 0 ]; then echo "VERDICT: feedback to address on $repo#$pr"; exit 0; fi
exit 1`
  - Output:
    ```
    VERDICT: no open PR for fabro/run/01KY1WHQKHVT77ZA62MMD7PKE2 (merged/closed?) — stop watching
    ```


# Address PR feedback

The `check` stage output (in your preamble) names the PR and ends with a `VERDICT:` line.

**If the verdict says the PR left draft, merged, closed, or has no open PR**: do nothing.
Reply with only this JSON: `{"preferred_next_label": "Done"}`

**Otherwise**, address the feedback:

1. Read ALL THREE comment sources — the top-level view alone silently misses line comments:
   - `gh pr view <n> --repo <repo> --comments` (top-level comments)
   - `gh api repos/<repo>/pulls/<n>/comments` (line-level review comments)
   - `gh api repos/<repo>/pulls/<n>/reviews` (review summaries — including requests on APPROVED reviews)
2. For each unresolved thread or comment newer than the head commit: implement what it asks in
   the working tree (current branch is the PR head branch). Run the checks the change touches
   (`cd apps/fabro-web && bun test && bun run typecheck` for web code; `cargo test -p <crate>`
   for Rust). Skip nothing silently — if you disagree with a comment, say so in the reply
   instead of implementing it.
3. Commit with a message referencing what feedback it addresses. Push the current branch:
   `gh auth setup-git` once, then `git push origin HEAD`.
4. Reply to every thread/comment you acted on, prefixed `[fabro]`, saying what you did (one or
   two sentences). Resolve each thread you addressed:
   `gh api graphql -f query='mutation($t:ID!){resolveReviewThread(input:{threadId:$t}){thread{id}}}' -f t=<thread-id>`
   (thread ids come from the reviewThreads GraphQL query). Do NOT post new top-level comments
   except replies; always prefix your comments `[fabro]` — the watcher filters on it.

When done, reply with only this JSON: `{"preferred_next_label": "Keep watching"}`
