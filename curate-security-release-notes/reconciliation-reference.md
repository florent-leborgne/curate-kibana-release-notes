# Missed-PR reconciliation reference

Use these recipes as starting points. Adjust version, dates, and branch names to the release.

## Candidate sets

### Security labels

Read the current label list from the generator before you query, because it changes:

```bash
gh api repos/elastic/kibana-release-notes/contents/src/config/templates/common.ts \
  --jq '.content' | base64 -d \
  | sed -n '/export const securityLabels/,/];/p'
```

### Version, release-note, and Security labels

GitHub label filters are conjunctive, so run one query per release-note type and filter for Security labels afterward:

```bash
for type in feature enhancement fix bug deprecation breaking; do
  GH_PAGER=cat gh pr list \
    --repo elastic/kibana \
    --state merged \
    --search "label:vVERSION label:\"release_note:${type}\"" \
    --limit 1000 \
    --json number,title,mergedAt,labels,url
done
```

Keep PRs that have at least one label from `securityLabels`. Keep the rest as a second list for the changed-paths check below. Confirm the repository's exact version-label format before running the query.

### Security changes without Security labels

For each PR in the second list, check whether it changes Security code:

```bash
GH_PAGER=cat gh pr view PR_NUMBER \
  --repo elastic/kibana \
  --json files \
  --jq '.files[].path' | grep -E '^x-pack/solutions/security/'
```

A match means the PR needs manual review, not that it belongs in the Security notes. Apply the scope rules in step 2 of the skill.

### Late merges

Use the initial generator date as the lower bound:

```bash
GH_PAGER=cat gh pr list \
  --repo elastic/kibana \
  --state merged \
  --search "merged:>=YYYY-MM-DD base:main" \
  --limit 1000 \
  --json number,title,mergedAt,baseRefName,labels,url
```

Filter the result for `release_note:*`, target-version, backport, and Security labels. Do not discard unlabeled candidates until their release scope is checked.

### Labels changed after generation

Find merged PRs updated after the exact generator cutoff:

```bash
GH_PAGER=cat gh pr list \
  --repo elastic/kibana \
  --state merged \
  --search "updated:>=YYYY-MM-DD" \
  --limit 1000 \
  --json number,title,mergedAt,updatedAt,labels,url
```

Filter `updatedAt` against the exact generator timestamp. For each remaining candidate, inspect version, release-note, and Security label events:

```bash
gh api --paginate repos/elastic/kibana/issues/PR_NUMBER/events \
  --jq '.[] |
    select(.event == "labeled" or .event == "unlabeled") |
    select(.label.name == "vVERSION"
      or (.label.name | startswith("release_note:"))
      or (.label.name | startswith("Team:"))
      or (.label.name | startswith("Feature:"))) |
    [.created_at, .event, .label.name] | @tsv'
```

Reconsider any PR whose target-version eligibility, release-note type, or Security routing changed after generation, even if it merged before the cutoff.

### Release branch and backports

```bash
git fetch origin main TARGET_BRANCH
git log --oneline PREVIOUS_RELEASE..origin/TARGET_BRANCH -- x-pack/solutions/security
```

For suspicious commits, identify the PR and original/backport relationship:

```bash
GH_PAGER=cat gh pr view PR_NUMBER \
  --repo elastic/kibana \
  --json number,title,baseRefName,mergedAt,labels,body,comments,files,url
```

Check `backport:*` and version labels, `ONMERGE` backport targets in the PR body, and original PRs that reached `main` after the generator cutoff. The path filter is a shortcut, not a complete list. Some Security changes live in shared packages or platform plugins.

### Later reverts and disabled changes

For every included PR, search later PRs in the release window for its number, title terms, and affected paths. Inspect PRs that revert, disable, or supersede the behavior even when they use `release_note:skip`. Remove an entry when the described behavior does not exist at the target release HEAD.

### Elastic Defend

Elastic Defend notes come from the `endpoint-dev` template in the generator. That repository is private.

- Ask the user for the latest Elastic Defend generator output for the target version.
- If you have access, confirm each change with `gh pr view PR_NUMBER --repo elastic/endpoint-dev`.
- Compare the Defend bullets in the target section with that output in both directions.
- Never add an `endpoint-dev` link to the published notes.

## Build-candidate verification

```bash
# 1. Latest BC's Kibana commit (authoritative cutoff)
curl -s https://artifacts-api.elastic.co/v1/versions/<version>/builds/<version>-<hash> \
  | jq -r '.build.projects.kibana.commit_hash'

# 2. Projects in the BC, to look for an Elastic Defend commit
curl -s https://artifacts-api.elastic.co/v1/versions/<version>/builds/<version>-<hash> \
  | jq '.build.projects | keys'

# 3. Candidate PR's backport merge commit on the Kibana release branch
git fetch origin <release-branch>
git log origin/<release-branch> --since=<window-start> --format='%H|%cI|%s' | grep -E '\(#<PR>\)'

# 4. Ancestry test: is the backport in the BC?
git merge-base --is-ancestor <backport-commit> <bc-kibana-commit> && echo IN || echo OUT
```

Add only `IN` results. Record `OUT` results as deferred and re-check them on the next update run against the next BC. Always test ancestry. Never use the merge timestamp.

## Comparison

Normalize each candidate to the original user-facing PR when possible. Record:

- PR number, or the Defend change title
- Original or backport PR
- Merge date
- Branch containing the change
- Version, release-note, and Security labels
- Expected file and section
- Present in `index.md`, `breaking-changes.md`, `deprecations.md`, or `known-issues.md`
- Resolution

```text
expected - actual = potentially missed
actual - expected = potentially stale or incorrectly included
```

Do not close the reconciliation with unexplained differences.

## Common failure modes

- The generator ran before a late merge.
- A version label was missing or added after generation.
- A Security PR had `release_note:*` but no label in `securityLabels`, so it went to the Kibana notes or to no notes.
- A new Security team or feature label was absent from `securityLabels`.
- A Kibana core change had a Security team label, so priority 100 routed it to the Security notes.
- A PR used the legacy `release_note:bug` label and landed in the uncategorized list.
- A backport merged without matching metadata on the original PR.
- A later `release_note:skip` PR reverted or disabled an included change.
- Elastic Defend output was regenerated after the Kibana output, and new Defend changes were never added.
- A fix in the target release resolved a known issue, but the known issue was not updated.
