# Curate Elastic Security release notes

An agent skill that turns Kibana and Elastic Defend release-note generator output into complete, accurate Elastic Security Stack release notes.

It removes Kibana core, Observability, Serverless-only, and internal-only entries, repairs labels and generator mappings, rewrites concise user-facing notes, updates known issues, and reconciles late or missed PRs against the latest build candidate.

This skill edits release-note files in `elastic/docs-content` under `release-notes/elastic-security/`. It is adapted from [curate-kibana-release-notes](https://github.com/florent-leborgne/curate-kibana-release-notes).

## Install

Clone or copy this folder into your agent skills directory:

```bash
ln -s "$PWD" ~/.claude/skills/curate-security-release-notes
```

## Use

```text
/curate-security-release-notes 9.3.0
```

If you omit the version, the skill asks for it before starting.

## Related skills

- `docs-kibana-release-notes` converts raw generator output into the Security notes format. Run it first, then use this skill to curate the result.
- `curate-kibana-release-notes` curates the Kibana core notes. Kibana core entries that this skill removes belong there.
- `docs-review-changelog` and `docs-fix-changelog` supply changelog quality and rewrite guidance used during the editorial pass.
