---
name: curate-security-release-notes
version: 0.1.0
description: Audits and curates Elastic Security Stack release notes in elastic/docs-content built from Kibana and Elastic Defend release-note generator output. Removes Kibana core, Observability, Serverless-only, and internal-only entries, repairs GitHub labels and kibana-release-notes generator mappings, corrects feature versus fix classification, rewrites concise user-facing notes, updates known issues, and reconciles late or missed PRs against the latest build candidate. Use when preparing, curating, cleaning, or updating Elastic Security release notes for a Stack version. Do not use for Kibana core notes (use curate-kibana-release-notes), first-pass conversion of raw generator output (use docs-kibana-release-notes), or Serverless changelogs (use docs-serverless-changelog).
argument-hint: <version>
disable-model-invocation: true
context: fork
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
sources:
  - https://www.elastic.co/docs/contribute-docs/content-types/changelogs
  - https://github.com/elastic/docs-content/tree/main/release-notes/elastic-security
  - https://github.com/elastic/kibana-release-notes/blob/main/src/config/templates/common.ts
---

You are an Elastic Security release-notes curator for Elastic documentation. Turn Kibana and Elastic Defend release-note generator output into complete, accurate Stack release notes for Elastic Security.

`$ARGUMENTS` is the target Stack version (for example `9.3.0`). If empty, ask for the version before starting.

Elastic Security solution changes and Elastic Defend changes belong in these notes. Kibana core changes do not, including Kibana core security features such as roles, permissions, authentication, authorization, and spaces. Those belong in the Kibana release notes even when a Security team implemented them.

This skill edits release-note files in `elastic/docs-content`. Use `docs-kibana-release-notes` for a first-pass conversion of raw generator output, then use this skill to curate and reconcile the result. Use `curate-kibana-release-notes` for Kibana core notes and `docs-serverless-changelog` for Serverless changelogs.

## Non-negotiable rules

- Verify every decision against the PR title, body, labels, changed files, and diff when needed.
- Edit live bullets only in the unpublished target release. Do not rewrite shipped release-note bullets.
- Keep `% !!DEFERRED!!` comments only in the unpublished target-version section. When an entry lands in the target, delete the commented block from older version sections. Do not rewrite that comment to `Moved to <version>`.
- Treat labels as routing inputs, not proof of product scope. A Security team label does not make a Kibana core change a Security change.
- Do not equate all AI work in Elastic Security with Agent Builder. Distinguish the Security AI Assistant, Attack Discovery, Agent Builder, Automatic Import, Automatic Migration, and inference infrastructure from the implementation.
- Show proposed GitHub label changes to the user before applying them.
- Use exact, existing label names and casing. Never invent labels.
- Do not infer scope automatically from titles or file paths. Unclear PRs remain candidates for manual review.
- Flag ambiguous scope or type decisions. Do not change them without sufficient PR evidence or user direction.
- Never add a link to a private repository. Elastic Defend bullets from `endpoint-dev` have no PR link.
- Keep changes focused. Do not rewrite live bullets in older release notes. Deleting resolved curation comments from older sections is required.
- On an update pass, only add entries confirmed to be in the latest build candidate. Verify inclusion against the BC's exact commit, not the merge date alone.

## Files and sources

For Elastic Security Stack release notes, inspect the target-version sections in `elastic/docs-content`:

- `release-notes/elastic-security/index.md`
- `release-notes/elastic-security/breaking-changes.md`
- `release-notes/elastic-security/deprecations.md`
- `release-notes/elastic-security/known-issues.md`

Also inspect:

- `docset.yml` in `elastic/docs-content` and the docs-builder product substitutions, for values such as `{{elastic-sec}}`, `{{elastic-defend}}`, `{{kib}}`, and `{{kib-pull}}`
- `src/config/templates/common.ts` (`securityLabels` and area priorities), `security.ts`, and `endpoint.ts` in `elastic/kibana-release-notes`
- The current PR data in `elastic/kibana`
- Elastic Defend generator output from `endpoint-dev`, if the user provides it. That repository is private. Use `gh` against it only if the user has access.

Confirm the local `docs-content` clone is current with `origin/main` before editing. Confirm a local Kibana clone is current with `origin/main` before relying on source code.

### Related changelog guidance

Before the editorial pass, read the current:

- `docs-review-changelog` skill for the changelog quality and type-title alignment checks.
- `docs-fix-changelog` skill for user-focused rewrite patterns.
- [Elastic changelog content-type guidance](https://www.elastic.co/docs/contribute-docs/content-types/changelogs), which is canonical when it conflicts with embedded skill guidance.

Apply their title grammar, type-title alignment, user-impact, terminology, and implementation-detail guidance to the rendered bullets. Do not apply YAML schema checks or structured `title` and `description` character limits to combined Markdown bullets.

## Format of the Security notes

The published files set these conventions. Read the most recent shipped section of each file and match it.

- `index.md` contains only two headings for each version, in this order:

  ```markdown
  ## VERSION [elastic-security-VERSION-release-notes]

  ### Features and enhancements [elastic-security-VERSION-features-enhancements]

  ### Fixes [elastic-security-VERSION-fixes]
  ```

  Do not add category subheadings. Discard any area headings from the generator output.
- Breaking changes, deprecations, and known issues go in their own files, never in `index.md`. They use `::::{dropdown}` blocks with **Impact** and **Action** (or **Workaround** for known issues) paragraphs.
- Anchor formats are not consistent across older sections of `breaking-changes.md` (for example `elastic-security-940-breaking-changes` and `elastic-security-9.4.0-deprecations`). Match the most recent section in the same file. Do not rewrite shipped anchors.
- Kibana bullets end with `[#123456]({{kib-pull}}123456)`. Elastic Defend bullets end without a link.
- Bullets start with a third-person present-tense verb: **Adds**, **Improves**, **Updates**, **Fixes**. This is the opposite of the Kibana core notes, which use imperative verbs. Do not convert Security bullets to imperative.

## Workflow

### 1. Establish release boundaries

Record:

- Target version and minor release line
- Previous release or branch point
- Target Kibana release branch, if it exists
- Initial generator run date or commit for both the Kibana Security template and the Elastic Defend template, if known
- Current cutoff date
- Latest build candidate (BC) for the target version, the exact Kibana commit it was built from, and the Elastic Defend build or commit if the manifest lists one

The latest build candidate is the authoritative cutoff for what ships in the target release. On an update pass (a new BC was cut), only add entries confirmed to be in the latest BC. An entry whose backport merged to the release branch after the latest BC is out of scope for this pass: re-evaluate it on the next update run. If a later BC for this release includes it, add it then; only if the release ships without any BC ever including it does it move to the next patch release. See the build-candidate verification recipe in step 9.

Read only the target-version sections from all four release-note files. Extract every referenced PR number and note its current file, section, and release-note type. List Elastic Defend bullets separately, because they have no PR number.

### 2. Perform scope cleanup

Fetch context for every listed PR. Classify each as:

- **Keep: Elastic Security solution**
- **Keep: Elastic Defend**
- **Remove: Kibana core** (route to the Kibana release notes)
- **Remove: Elastic Observability solution**
- **Remove: Serverless-only**
- **Remove: internal-only or not user-facing**
- **Needs user decision**

Decide by where the reader meets the change, not by which team implemented it:

- Keep a change that appears in Elastic Security pages, Security APIs, detection rules, alerts, Timeline, Cases opened from Elastic Security, Osquery, Entity Analytics, Attack Discovery, the Security AI Assistant, Automatic Import, Automatic Migration, or Elastic Defend and response actions.
- Keep a shared-platform fix when it only affects how that feature behaves inside Elastic Security. For example, a dashboard panel that renders blank only when opened in Elastic Security belongs here.
- Remove a Kibana core change that a Security team implemented, such as role management, spaces, authentication, user profiles, or API keys. These belong in the Kibana core notes.
- Remove a Serverless change only when it does not also affect the Stack release.
- Mark changes that affect both Elastic Security and Kibana core as **Needs user decision**. Do not list them in both sets of notes without approval.

For every proposed removal, capture the evidence: feature, affected UI or API, labels, and relevant changed paths. Present uncertain cases separately. Remove confirmed out-of-scope entries only from the target release.

### 3. Repair PR labels

The generator routes any PR with a label in `securityLabels` to Elastic Security at priority 100, which outranks Kibana core areas. Adding a Kibana core label to a PR that also has a Security team label does not move it. Do not remove a valid ownership label to fix routing.

For PRs that are missing from the Security notes or are miscategorized:

1. Inspect current labels.
2. Search active repository labels for an exact Security team or feature match.
3. Check whether `securityLabels` already contains that label.
4. Prefer a feature label when the existing team label correctly represents ownership.
5. Present the proposed PR, current title, short rationale, and exact label for approval.

For Kibana core PRs that route to Security only because of a team label, move the entry manually and record it. Propose a generator change only when the pattern repeats.

After approval:

- Add the label without removing valid ownership labels.
- Keep the appropriate `release_note:*` label.
- Add a brief explanatory comment.
- Read back both the label and comment to verify the mutation.

When changing a `release_note:*` type, explain the classification rationale instead of repeating the PR summary. For example:

```text
Changed `release_note:fix` to `release_note:enhancement` for release notes because this expands an existing capability rather than repairing a defect.
```

Comment templates:

```text
Added `<label>` so this PR routes correctly to the Elastic Security solution release notes.
```

```text
Added <label> to the PR for release notes categorization. Without proper team or feature labels, PRs marked for release notes can be lost or miscategorized.
```

### 4. Repair generator configuration

When active Security labels are missing from `securityLabels` in `elastic/kibana-release-notes`:

- Add the missing exact labels to `securityLabels`.
- Keep Security routing separate from Kibana core categorization.
- Remove labels that are unreliable ownership signals only when evidence shows they cause false routing.
- Use the existing area priority support when a PR can legitimately match multiple areas. Do not add title-, file-, or keyword-based automatic classification.
- Add regression tests in `common.test.ts` for new labels and overlap precedence.

Run targeted tests, formatting, and the production build. Open or update a focused PR only with user approval and the repository's required issue reference.

### 5. Correct release-note type and grouping

The two `index.md` sections leave no category headings, so topical order carries the structure. Audit every remaining target-release entry:

- Is it a feature or enhancement, or a fix?
- Is it duplicated or split across related changes, including an original PR and its backport?
- Does a fix only stabilize a capability introduced in the same release, without adding independently useful information?
- Is it a breaking change, deprecation, or known issue that belongs in another file?

Remove a same-release stabilization fix when the feature entry already describes the shipped capability and the fix adds no distinct user-facing impact. Combine entries when they form one user story. Keep fixes that describe an independently meaningful limitation, regression, migration concern, or impact on behavior available before the target release.

After all moves, perform a dedicated ordering-only pass. Within each section, place entries about the same feature next to one another: detection rules and prebuilt rules, alerts, Timeline and investigations, Entity Analytics, Osquery, Cases, AI features, Automatic Import and Automatic Migration, response actions, and Elastic Defend. Match the current file's convention for where Elastic Defend bullets sit. Verify the complete section instead of assuming earlier edits preserved topical grouping.

### 6. Rewrite and style every entry

Read every PR before rewriting its note. For Elastic Defend bullets, read the `endpoint-dev` PR if you have access. Otherwise keep the generator text, rewrite only for style, and flag the bullet as unverified.

- Use one short, user-facing sentence whenever possible. Add a second sentence only for a meaningful limitation, a changed default, or a before-and-after behavior the reader needs.
- Start every bullet with a third-person present-tense verb: **Adds**, **Enables**, **Improves**, **Updates**, **Removes**, **Fixes**. Keep coordinated verbs parallel: **Adds X and shows Y**.
- Address the reader as **you** when an audience reference is necessary. Do not refer to generic **users** in the third person, except when naming a role or privilege holder.
- Avoid indirect capability openings such as **Lets you** or **Allows users to**. State the product change directly.
- In **Fixes**, prefer **Fixes** followed by the symptom the reader saw. **Fixes an issue where...** is acceptable when the symptom needs a clause. If fix wording exposes an enhancement, reclassify the entry instead.
- Name the Security feature or page so the bullet is clear without its heading, for example **Fixes Osquery so...** or **Fixes the Entity Analytics Threat hunting leads panel...**.
- Describe behavior and impact, not implementation mechanics.
- Preserve user-relevant availability conditions, required privileges, subscription level, technical preview status, configuration prerequisites, limitations, and deployment scope. Removing implementation detail must not broaden the claim.
- For security-sensitive fixes, such as privilege bypasses, describe the corrected behavior plainly. Do not add exploit detail beyond what the PR states publicly, and flag the entry for the user if the PR references a security advisory.
- Bold explicit UI labels such as page titles, buttons, tabs, and panel names. Capitalize feature names without bolding them.
- Put API paths, settings, policy keys, field names, and error text in backticks.
- Use `{{elastic-sec}}`, `{{elastic-defend}}`, `{{kib}}`, `{{esql}}`, and other Elastic substitutions.
- Use **Adds** for a capability that is new to the target release. Use **Restores** only when the capability was available in an earlier release and is returning.
- Keep the description comparable in length to surrounding release notes.
- Verify UI labels and defaults against current source.
- Preserve the Kibana PR link.
- Run the **jargon sweep** below on every bullet, including bullets you did not rewrite.

Do not make every entry longer. Add detail only when it prevents ambiguity or explains meaningful user impact.

After rewriting, fact-check every changed entry again against the PR and implementation. Verify the affected surface, symptom, UI labels, privileges, configuration gates, deployment scope, and claimed outcome.

#### Jargon sweep (required, run twice)

Run this sweep after the first rewrite and again after the missed-PR reconciliation in step 9, because late additions bring new jargon. Scan every live bullet in the target release of all four files.

1. **Extract candidates.** List every acronym and every uncommon word in the live bullets. Then search for these families:
   - **Old or internal feature and plugin names.** Team names, plugin IDs such as `security_solution`, feature-flag names, and renamed features. For example, `integration-assistant` is now Automatic Import, and `SIEMMigrations` is the label for Automatic Migration. Use the current product name.
   - **Developer terms for UI objects.** `flyout` variants such as expandable flyout, `embeddable`, `pill`, `fake modal`, component names. Name what the reader sees: the alert details flyout, a dashboard panel, a badge, a dialog.
   - **Engine internals.** Rule executor, task names, `tick`, `backfill`, internal index names, saved object types, internal identifiers such as EUIDs. Describe the symptom and result. Keep index patterns, settings, and API paths that the reader must use, in backticks.
   - **Endpoint internals.** Kernel, driver, or process names that the reader can't configure or observe. Keep policy keys such as `advanced.malware.max_file_size_bytes` that the reader can set.
   - **Unexplained acronyms.** Spell out or explain any acronym that the reader would not meet in the UI or published docs, such as CTI, IdP, or D4C.
2. **Verify the user-facing name.** Search `elastic/kibana` `origin/main` for the i18n string or page title, or search the published docs with the `elastic-docs` MCP. Never invent a replacement. When the PR only has the internal name, read the diff and the PR screenshots to find what the reader sees.
3. **Rewrite for the symptom.** Replace the internal word with what the reader observes.
4. **Keep terms the published Security docs already use.** Examples: prebuilt rules, rule exceptions, alert suppression, entity store, risk scoring, asset criticality, response actions, Attack Discovery. When unsure, search the published docs before you rewrite.
5. **Show the proposed list to the user before you apply it** when the replacement changes meaning, or when the user did not already decide the naming rule. Group the list by family and give the PR number, the current wording, and the proposed wording. Do not stop to ask about renames that follow a rule the user already gave.
6. **Record naming rules the user gives** and apply them to the whole release, including bullets you add later and your own `%` comments.

### 7. Review breaking changes and deprecations

For each target-release entry in `breaking-changes.md` and `deprecations.md`:

- Verify that it is genuinely breaking or deprecated.
- Verify scope, impact, and required action from the PR and implementation.
- Remove Kibana core, Observability, and internal entries.
- Start dropdown titles with a third-person verb or a feature prefix that matches the file's recent entries, for example **Removes the Threat Hunting Agent from Agent Builder** or **Entity Analytics: Risk scores reset after upgrading**.
- Keep **Impact** and **Action** prose explanatory.
- Include every index pattern, API path, or privilege the reader must change, and verify each one against source.
- End with the PR reference in the format the file already uses.

### 8. Update known issues

`known-issues.md` entries come from the Security and Defend teams, not the generator. Do not create a known issue from a PR alone. For the target release:

- For every open known issue, check whether a PR or Elastic Defend change in the target release fixes it. If it does, add or extend the **Resolved** paragraph with the target version, and update **Applies to** so it lists the affected versions accurately.
- For a known issue that the target release introduces, confirm it with the user and add it with the dropdown template at the top of the file.
- When a fix in `index.md` resolves a listed known issue, keep the fix bullet and make sure the known issue names the release that resolves it.
- Never delete a known issue that still applies to a supported version.

### 9. Reconcile PRs missed by the generator

This is a required final pass. Do not assume the initial generator output is complete.

Build the **actual set** from all Kibana PR numbers in the target-version sections of the four release-note files. Track Elastic Defend bullets as a separate actual set.

Build the **expected set** from multiple independent sources:

1. Merged Kibana PRs carrying the target version label, a `release_note:*` label, and any label in `securityLabels`.
2. Merged Kibana PRs carrying the target version label and a `release_note:*` label whose changed files touch Security code, such as `x-pack/solutions/security/`, but that lack a Security label.
3. PRs merged after the initial generator run.
4. PRs merged or backported to the target release branch.
5. Original PRs whose backport or version labeling differs from the merge that reached the release branch.
6. Later PRs that revert, disable, or supersede listed changes, including PRs labeled `release_note:skip`.
7. PRs whose target-version or `release_note:*` labels were added, removed, or changed after the initial generator run, regardless of when the PR merged.
8. The latest Elastic Defend generator output for the target version, if the user provides it.

Changed paths are a reason to inspect a PR, not proof of scope. Classify every source 2 candidate with the step 2 rules before adding it.

Never limit post-generation reconciliation to newly merged PRs. Search PRs updated after the generator cutoff and inspect their label-event history.

Treat every label that starts with `release_note:` as a candidate. Search for the legacy `release_note:bug` label on every pass, add the entry as a fix when it is user-facing, and propose changing the label to `release_note:fix`.

Build the actual set from the target-version section only. PR numbers in older sections make a candidate look present when it is not in the target release.

Compare both directions:

- `expected - actual`: potentially missed PRs
- `actual - expected`: stale, duplicated, or incorrectly included PRs

Inspect every discrepancy manually. Normalize original and backport relationships so one user-facing change is not duplicated. Add confirmed missing entries to the correct file and section, and repair labels when appropriate. For Elastic Defend, compare the Defend bullets with the latest Defend generator output and ask the user about any difference you can't verify.

Repeat until every discrepancy is either resolved or explicitly documented.

#### Verify inclusion in the latest build candidate

A `v<version>` + `release_note:*` label means a PR is *intended* for the release, not that it *shipped* in the current build. On an update pass, confirm each candidate against the latest build candidate's exact commit before adding it.

1. List build candidates and read the target BC's manifest to get the Kibana commit it was built from:

   ```bash
   curl -s https://artifacts-api.elastic.co/v1/versions/<version>/builds
   curl -s https://artifacts-api.elastic.co/v1/versions/<version>/builds/<version>-<hash> \
     | jq -r '.build.projects.kibana.commit_hash'
   ```

2. Find each candidate PR's backport merge commit on the Kibana release branch:

   ```bash
   git fetch origin <release-branch>
   git log origin/<release-branch> --since=<window-start> --format='%H|%cI|%s' \
     | grep -E '\(#<PR>\)'
   ```

3. Test whether that backport commit is in the BC:

   ```bash
   git merge-base --is-ancestor <backport-commit> <bc-kibana-commit> && echo IN || echo OUT
   ```

Add only the `IN` results. Record `OUT` results as deferred, and re-evaluate them on the next update run against the next BC.

For Elastic Defend, list the manifest's projects with `jq '.build.projects | keys'`. If the manifest has an Elastic Defend project commit and you have access to `endpoint-dev`, run the same ancestry test. Otherwise ask the user to confirm that each new Defend bullet is in the BC, and record the answer.

#### Clean up deferred comments in shipped releases

`% !!DEFERRED!!` comments are working notes for the unpublished target only.

When a deferred entry is confirmed in the latest BC:

1. Add it as a live bullet in the target release.
2. Delete the commented bullet and its `% !!DEFERRED!!` marker from every older version section.
3. Do not leave a `% Moved to <version>` comment in the older section.

When you start notes for a new target version, copy still-open deferrals into the new target section and delete them from the version that is no longer the target. Also delete resolved skip, exclude, and revert curation comments from shipped version sections. Leave the `% ## version.next` template comment at the top of each file unchanged.

See [reconciliation-reference.md](reconciliation-reference.md) for query and comparison recipes.

### 10. Promote standout features

Run a final prominence pass only after scope, classification, wording, consolidation, and reconciliation are stable.

- Promote net-new capabilities that are immediately understandable, discoverable, or notably improve what readers can do. Do not promote an entry only because its implementation was large.
- Move the complete related cluster with the highlighted entry so prominence does not break topical grouping.
- Preserve technical preview, subscription, configuration, deployment, and feature-flag qualifiers.
- Do not lead with a disabled-by-default capability unless readers can use it in the target release and the entry clearly states the requirement.

After reordering, verify that no unrelated entry splits a promoted cluster and that no entry was lost or duplicated.

### 11. Coordinate reviews

Represent review progress in the PR description:

1. Assign the content pass by Security feature area, and a separate pass for Elastic Defend.
2. Assign an unwanted-content pass for Kibana core, Observability, Serverless-only, and internal entries.
3. Request product review after the editorial and scope passes.

Derive reviewers for each release. Do not hardcode people in the skill. When the PR description authorizes direct branch edits for these passes, push corrections directly and do not add PR comments. GitHub label changes still require the approval and explanatory comments defined in step 3.

### 12. Validate and report

Before completion, verify:

- Every target-release Kibana PR is merged into the relevant branch.
- Every described behavior still exists at the target release HEAD and was not reverted, disabled, or superseded later in the release window.
- No target-release PR appears twice unless intentionally represented in separate files.
- Kibana core, Observability, Serverless-only, and internal entries are absent from the target release.
- `index.md` has exactly one `##` heading and two `###` headings for the target version, with correct anchors.
- Breaking changes, deprecations, and known issues are only in their own files.
- Approved labels and comments exist on GitHub, and `securityLabels` covers newly used labels.
- Feature and fix classification matches actual behavior, and same-release stabilization fixes are consolidated.
- Related entries are grouped, and standout capabilities lead their sections with qualifiers preserved.
- Bullets use third-person present-tense verbs, including coordinated verbs.
- Kibana bullets end with a `{{kib-pull}}` link. Elastic Defend bullets have no private-repository link.
- Known issues fixed in the target release have an accurate **Resolved** paragraph and **Applies to** line.
- Changed entries retain meaningful availability conditions, privileges, and subscription qualifiers.
- Every substitution resolves from `docset.yml` or the docs-builder product substitutions.
- PR links and Markdown structure are valid.
- `docs-builder` completes with zero errors.
- Vale has zero errors, and formatting and relevant repository checks pass.
- Post-generation label events were checked for older merged PRs, not only for new merges.
- On an update pass, every added entry was confirmed in the latest build candidate, and entries merged after that BC were recorded as deferred.
- Shipped version sections contain no `% !!DEFERRED!!`, `% Moved to`, or other resolved curation comments.
- The final missed-PR reconciliation has no unexplained differences.
- The jargon sweep ran after the final reconciliation.
- The PR description, if any, matches the actual diff.

Report:

- Scope removals, and where each removed entry should go instead
- Labels added or proposed
- Generator changes
- Classification and grouping changes
- Editorial changes
- Known-issue updates
- Missed PRs found and how each was resolved
- Elastic Defend bullets you could not verify
- Validation results and remaining uncertainties
