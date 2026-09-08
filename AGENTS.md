# AGENTS.md

Guidance for AI coding agents (Claude Code, and anything else reading `AGENTS.md`) working in
this repository. `CLAUDE.md` imports this file.

## What this repo is

A single deliverable: `.coderabbit.yaml` (~1560 lines), the CodeRabbit organization config for
the **coreruleset** GitHub org. There is no application code, no build, and no test suite. Every
change is an edit to that one YAML file.

Schema: `https://coderabbit.ai/integrations/schema.v2.json` (declared in the file's first line;
that URL 301-redirects — fetch `https://storage.googleapis.com/coderabbit_public_assets/schema.v2.json`).
Reference: https://docs.coderabbit.ai/configure-coderabbit/

## Validating a change

```bash
yamllint -d relaxed .coderabbit.yaml   # only line-length warnings are expected
uvx check-jsonschema --schemafile <schema.json> .coderabbit.yaml
```

`.github/workflows/validate-coderabbit.yml` runs both on every PR touching the config, and weekly
(the schema is fetched at run time, so upstream changes can invalidate a config that already
merged). yamllint is deliberately not `--strict`: line-length warnings must not fail, but a
duplicated key still exits 1.

Schema caps are enforced silently *at review time* — CodeRabbit truncates rather than erroring —
but every one is a `maxLength`/`maxItems` in the schema, so `check-jsonschema` does catch them:
`tone_instructions` ≤ 250 chars · `labeling_instructions[].instructions` ≤ 3000 ·
`path_instructions[].instructions` ≤ 20,000 · `custom_checks[].name` ≤ 50 ·
`custom_checks[].instructions` ≤ 10,000 · `custom_checks` ≤ 50 items ·
`linked_repositories` ≤ 20 items · `linked_repositories[].instructions` ≤ 2000.

## Two kinds of repo, one config

The org holds the rule set (`coreruleset`, `plugins/*`, `coraza-coreruleset`: seclang `.conf`,
regex-assembly `.ra`, go-ftw YAML) **and** its tooling (`go-ftw`, `crs-toolchain`, `albedo`,
`rassemble-go`, `libinjection-go` in Go; `crs-linter`, `msc_pyparser` in Python). Path-scoped
guidance is what keeps each from being reviewed as the other — this is the main design
constraint on the file.

## Where review logic belongs

- **`reviews.path_instructions`** — per-file, glob-scoped. Use when the finding needs
  *line-level localization*. Nine entries. The four seclang paths are deliberately split rather
  than collapsed into `**/*.conf`, because each is a different review job: `rules/*.conf`
  (detection rules), `plugins/*.conf` (reserved ID range, before/after/config file ordering),
  `crs-setup.conf.example` (shipped defaults, not patterns), `rules/*.conf.example` (exclusion
  templates that must stay inert). Then `**/*.ra`, `**/tests/**/*.yaml` (go-ftw), `**/*.go`,
  `**/*.py`, `**/.github/workflows/**`.
- **`reviews.pre_merge_checks.custom_checks`** — runs once per PR against the whole diff. Use
  for cross-file invariants and policy a single line can't express. The CRS-specific ones:
  regex-assembly source-of-truth (`.conf` `@rx` edited without its `.ra`), go-ftw test coverage,
  ReDoS/RE2 compatibility, false-positive risk & existing coverage, rule metadata & ID
  conventions, rule/config breaking changes, AI contribution disclosure. Plus the generic
  supply-chain and appsec checks.

The domain knowledge in these strings is derived from `../coreruleset/CLAUDE.md` (rule ID
ranges, paranoia levels, `.ra` gotchas, ReDoS shapes, PL4 coverage probe, crs-linter invocation,
PR conventions). Keep the two in sync when either changes.

## Conventions for checks

- Finding format is fixed file-wide: `⚠️ WARNING: <description> — <fix>`, with `❌ blocker`
  reserved for RE2-incompatible regexes, plugin rules in a core ID range, and hand-edited
  compiled regex that pre-commit.ci will revert.
- Every check is `mode: warning`. Escalating to `mode: error` requires also setting
  `request_changes_workflow: true`, and should follow a validated low false-positive rate on
  real CRS PRs.
- Ecosystem- and rule-scoped checks open with a **trigger condition** that returns `Passed` +
  "not applicable" when the diff touches nothing relevant — this avoids `Inconclusive` noise.
  Match that pattern in new checks.
- Checks describe **acceptable patterns** alongside violations so the reviewer has a target state.

## Things that will bite

- **Labeling lives in coreruleset/coreruleset's own `.coderabbit.yaml`, not here.** `release:*`
  is the changelog taxonomy consumed by `.github/release.yml` — it groups the generated release
  notes by these and excludes `release:ignore` entirely. They loosely mirror conventional-commit
  types, exactly one per PR (enforced via `mutually_exclusive_groups`), and the fallback is
  `release:ignore`, never "no label". Topic labels (`:mage: regex-assembly`, `:bomb: sqli`, …)
  are independent of that family. Do not invent `release:` values; the eight are exactly the
  ones that exist in coreruleset/coreruleset, and each maps to one `.github/release.yml`
  category. This config used to define `labeling_instructions` org-wide on the assumption that
  a missing label silently no-ops; that assumption was false — `go-ftw`, `crs-toolchain`, and
  `plugin-registry` already carry labels with the exact same literal names (`release:fix`,
  `release:ignore`, `release:new-feature`, `:book: documentation`) for unrelated purposes, so
  the org-wide config was actively mislabeling PRs there. The schema has no per-repository
  scoping field, so labeling was moved to a repo-local `.coderabbit.yaml` in
  coreruleset/coreruleset (`inheritance: true`, so it still layers on this org config). Label
  names are literal, emoji shortcode included (`":heavy_plus_sign: False Positive"`), and must
  already exist in that repo — CodeRabbit never creates them.
- **`inheritance: true`** is set. List fields (`custom_checks`, `path_filters`,
  `path_instructions`, …) are **replaced, not merged**, when a repo defines the same key
  locally; the repo must set `inheritance: true` in its own `.coderabbit.yaml` to layer rather
  than replace. Adding a list entry here only reaches repos that haven't overridden that key.
- **`path_filters` does not read `.gitignore`** — every exclusion must be listed explicitly.
- **`semgrep` and `opengrep` overlap** and would double-flag; exactly one stays enabled
  (opengrep).
- `base_branches` includes `lts/v.*` — backport PRs target LTS branches directly and must still
  be reviewed.
- `allow_non_org_members: true` and `enable_free_tier: true` are deliberate: CRS takes drive-by
  community contributions.
- **`linked_repositories` is capped at 5 by our CodeRabbit plan** (the schema itself allows up
  to 20). The five kept, chosen for the checks that fire across the whole org rather than one
  repo: `coreruleset` (rule IDs, approved tags, PL policy — everything else is downstream of
  it), `crs-toolchain` (`.ra` directive semantics, backing the regex-assembly source-of-truth
  check), `crs-linter` (what actually fails CI), `go-ftw` (the test runner every repo's tests
  depend on), and `documentation` (the counterpart-obligation check: flag a PR that changes
  documented behavior without a matching docs change). `documentation` carries the longest
  entry on purpose: it is the source of truth for everything operators are told (configuration,
  PL guidance, exclusion recipes, upgrade notes), so it outranks anything inferred from the
  rules when the question is what a user is supposed to do. Note it was renamed from
  `coreruleset-documentation`; GitHub redirects, but the config uses the canonical name.
  Dropped to fit the cap: `ftw-tests-schema` (overlaps `go-ftw`), `plugin-registry` (only backs
  the plugin ID-range check, scoped to plugin repos), `actions` and `renovate-config` (only
  relevant to workflow/dependency-policy discussions). Revisit if plugin repos start seeing
  frequent drive-by PRs — `plugin-registry` backs the one blocker-severity check among the four
  that were cut. Deliberately excluded regardless of the cap: `template-plugin` and
  `modsecurity-crs-docker` — they consume the rule set but nothing breaks across the edge.
