# drupal-contrib-upgrade-auditor

A GitHub Copilot agent skill that performs deep, multi-step impact analysis before upgrading a Drupal contributed (contrib) module in a brownfield codebase.

## What it does

When you ask Copilot to audit a contrib module upgrade, this skill runs a structured analysis across the codebase and produces a Markdown report saved to `.copilot_workspace/`. The report covers:

- **Metadata and dependency conflicts** — mismatches in `*.info.yml` and `composer.json` between your custom modules and the new contrib version.
- **Config schema and update hook risk** — schema keys that were removed or renamed (which will break `drush config:import`) and any `hook_update_N()` functions in the new release that will run on `drush updatedb`.
- **Code impact ("X-Ray")** — every custom file that uses a PHP class, service, hook, or plugin that changed in the new version, with a complexity estimate per finding.
- **Ghost dependencies** — custom code that calls into the module without declaring it as a dependency in metadata.
- **Test coverage inventory** — existing Behat and PHPUnit tests that cover the module's functionality and must be re-run after the upgrade.
- **Diff evidence** — the exact compare URL, tags, commit SHAs, or release archives used as the source of truth.
- **Patch overlay** — every `cweagans/composer-patches` patch targeting the module, assessed for whether it is still required, already upstreamed, or now conflicting.
- **Developer handoff references** — a self-contained set of facts, file paths, and code symbols so a developer or downstream agent can act without re-running discovery.

## When to use it

Trigger this skill by asking Copilot questions like:

- "Audit the impact of upgrading `drupal/webform` from `6.2.3` to `6.3.0`."
- "Check for breaking changes before I bump `drupal/paragraphs` to `2.0`."
- "What will break if I update `drupal/domain`?"
- "Evaluate the impact of moving `drupal/search_api` from `1.28` to `1.35`."

This skill is also invoked automatically by `drupal-core-upgrade-auditor` for each contrib module that needs a version bump during a Core upgrade.

## When NOT to use it

- Installing a module for the first time — no upgrade impact to analyze.
- General "how do I update a module" questions.
- Full Drupal Core version upgrades — use `drupal-core-upgrade-auditor` instead.

## Analysis phases

The skill uses a fast-fail triage gate before committing to the full analysis:

| Version bump type | Phases run |
| :--- | :--- |
| **Patch** (e.g., `2.1.3 → 2.1.5`) | Patch overlay + changelog scan only |
| **Minor** (e.g., `2.1.x → 2.3.x`) | Phases 1–2 + targeted Phase 3 |
| **Major** (e.g., `1.x → 2.x`) | All phases in full |
| **Non-SemVer / dev branch** | Treated as major risk |

### The full phases

| Phase | Name | What it does |
| :--- | :--- | :--- |
| 0 | Triage | Classifies the bump; announces scope to the user before proceeding |
| 1 | Metadata & Dependency Alignment | Composer dry-run, version comparison, custom metadata audit, patch inventory, config schema diff, update hook detection |
| 2 | Ghost Dependency & Surface Analysis | Namespace scan, service ID search, hook/plugin audit, test coverage inventory |
| 3 | Breaking Change Detection | Resolves upstream refs, fetches diff, audits API surface changes, re-checks patch overlay |

## Output

Reports are saved to:

```
.copilot_workspace/contrib-upgrade-audit-{module_name}-{from_version}-to-{to_version}.md
```

If the file already exists, a date suffix is appended to avoid overwriting prior runs.

## Relationship to other skills

| Skill | Relationship |
| :--- | :--- |
| `drupal-core-upgrade-auditor` | Orchestrates this skill — invokes it per flagged contrib module during a Core upgrade |
| `drupal-issue-queue` | Called during Phase 1 to check the module's issue queue for known breaking changes |
| `drupal-contribute-fix` | Use this if the skill surfaces an upstream bug that needs a patch or contribution |
