# Curate Kibana core release notes

An agent skill that turns Kibana release-note generator output into complete, accurate release notes for Kibana core applications.

It removes Observability and Security solution entries plus internal-only items, repairs labels and generator mappings, rewrites concise user-facing notes, and reconciles late or missed PRs against the latest build candidate.

This skill edits release-note files in `elastic/kibana`. It is not for converting generator output into Observability or Security Stack notes in `docs-content`.

## Install

```bash
npx --yes skills@latest add florent-leborgne/curate-kibana-release-notes -g
```

Or clone this repository into your agent skills directory:

```bash
git clone https://github.com/florent-leborgne/curate-kibana-release-notes.git ~/.claude/skills/curate-kibana-release-notes
```

## Use

```text
/curate-kibana-release-notes 9.3.0
```

If you omit the version, the skill asks for it before starting.

## Related skills

These catalog skills in [elastic/elastic-docs-skills](https://github.com/elastic/elastic-docs-skills) cover adjacent work:

- `docs-kibana-release-notes` converts generator output into Observability or Security Stack notes
- `docs-serverless-changelog` converts generator output into Serverless changelog format
- `docs-review-changelog` and `docs-fix-changelog` supply changelog quality and rewrite guidance used during the editorial pass
