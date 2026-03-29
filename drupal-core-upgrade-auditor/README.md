# drupal-core-upgrade-auditor

A GitHub Copilot agent skill that performs deep, multi-step impact analysis before upgrading Drupal Core in a brownfield codebase. It acts as an **orchestrator** — scanning the full site for compatibility issues and delegating per-module deep analysis to `drupal-contrib-upgrade-auditor`.

## What it does

When you ask Copilot to audit a Core upgrade, this skill produces a comprehensive upgrade roadmap saved to `.copilot_workspace/`. The report covers:

- **Core change record surface** — every Drupal.org change record tagged "Action needed" or "Behavior change" in the upgrade range that applies to this codebase, with risk ratings.
- **Custom code deprecation report** — every deprecated or removed API found in `web/modules/custom/` and `web/themes/custom/`, with file, line, deprecated symbol, replacement, and whether Rector can fix it automatically.
- **Contrib module compatibility matrix** — every enabled contrib module classified as Compatible, Needs update, No stable release, or Abandoned, with the action required for each.
- **Core patch overlay** — every `cweagans/composer-patches` patch targeting `drupal/core`, assessed for whether it is still required and whether it applies cleanly to the new version.
- **Update hook and config schema risk** — Core update hooks that will run on `drush updatedb` and config schema changes that will affect `drush config:import` against your `config/sync/` exports.
- **Recommended upgrade sequence** — a concrete, ordered list of steps a developer can follow from first code fix to final smoke test.
- **Developer handoff references** — all artifacts (change record links, `drupal-check` output, `drupal-contrib-upgrade-auditor` report paths, patch entries) needed to begin implementation without re-running discovery.

## When to use it

Trigger this skill by asking Copilot questions like:

- "Audit the impact of upgrading Drupal Core from `11.1` to `11.3`."
- "What needs to change before we move from Drupal 10 to Drupal 11?"
- "Which contrib modules aren't compatible with Drupal 11.2?"
- "Triage the risk of applying this Core security release."
- "Produce an upgrade roadmap for our Core bump."

## When NOT to use it

- Upgrading a single contrib module without changing Core — use `drupal-contrib-upgrade-auditor` instead.
- General "how do I update Drupal" questions.
- Security release confirmation with no API changes — the skill detects this in Phase 0 and exits early with a simple recommendation.

## Analysis phases

The skill uses a fast-fail triage gate. A security/patch release with no API changes exits immediately with a simple recommendation. All other bump types proceed through the full workflow:

| Phase | Name | What it does |
| :--- | :--- | :--- |
| 0 | Triage | Classifies the bump (security/minor/major/cross-PHP); announces scope; exits early for clean security releases |
| 1 | Core Change Record & Release Notes Review | Fetches Drupal.org change records and release notes for every version in the upgrade range; produces the change surface list |
| 2 | Custom Code Deprecated API Scan | Runs `drupal-check`, optional Rector dry-run, and targeted `grep_search` for change-record symbols across all custom code |
| 3 | Contrib Module Compatibility Audit | Builds full enabled-module list; classifies each module; invokes `drupal-contrib-upgrade-auditor` for every "Needs update" module; checks Core patches |
| 4 | Update Hook & Config Schema Risk | Identifies Core update hooks and config schema changes that require manual attention before `drush updatedb` and `drush config:import` |

## Output

Reports are saved to:

```
.copilot_workspace/core-upgrade-audit-{from_version}-to-{to_version}.md
```

If the file already exists, a date suffix is appended to avoid overwriting prior runs.

Each contrib module that needs a version update will also produce its own report via `drupal-contrib-upgrade-auditor`:

```
.copilot_workspace/contrib-upgrade-audit-{module_name}-{from_version}-to-{to_version}.md
```

## Skill composition

This skill is the top-level orchestrator in a two-skill upgrade pipeline:

```
drupal-core-upgrade-auditor        ← start here for Core upgrades
  └─ drupal-contrib-upgrade-auditor  ← automatically invoked per flagged module
```

| Skill | Relationship |
| :--- | :--- |
| `drupal-contrib-upgrade-auditor` | Delegate — invoked for each contrib module classified as "Needs update" |
| `drupal-issue-queue` | Called during Phase 3 to check module issue queues for compatibility status |
| `drupal-contribute-fix` | Use this if a contrib module has no stable release and needs an upstream patch or local fix |
