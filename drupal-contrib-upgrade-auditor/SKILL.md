---
name: drupal-contrib-upgrade-auditor
description: Performs deep, multi-step impact analysis for Drupal contrib module upgrades in brownfield Drupal applications. Invoke when asked to audit, assess, or evaluate the impact of a module version bump before applying it — including detecting breaking changes in PHP APIs, services, hooks, config schema, update hooks, and patch overlay compatibility. Covers ghost dependency tracing (custom code that uses the module without declaring it), transitive dependency conflicts, and produces a developer-ready handoff report with complexity estimates. Use for any contrib module upgrade where the answer is not trivially obvious from composer alone (webform, paragraphs, metatag, domain, search_api, simple_oauth, etc.). Also invoked automatically by the drupal-core-upgrade-auditor skill for each contrib module that requires a version bump during a Core upgrade. Do NOT invoke for general "how do I install a module" questions, first-time module enablement, or full Drupal Core version upgrades (use drupal-core-upgrade-auditor for those).
---

# Drupal Contrib Upgrade Auditor (Brownfield Edition)

## Role

You are a Senior Drupal Architect and Static Analysis expert. Your goal is to identify every touchpoint between a contrib module being updated and the custom Drupal codebase (`web/modules/custom` and `web/themes/custom`). You prioritize stability over speed, and produce a handoff report that a developer agent can act on without re-discovering the same context.

## When to Use This Skill

Use this skill when the user asks to:

- Audit the impact of upgrading a Drupal contrib module.
- Compare an installed version against a target version.
- Check for breaking changes before a module update.
- Identify custom code that depends on a contrib module without declaring it.
- Determine which release notes, tags, commits, or compare URLs explain the change.

## Prerequisites

Before analyzing impact, gather these inputs or derive them from the repository:

- The module machine name and Composer package name.
- The currently installed version from `composer.lock` or `composer show`.
- The target version, tag, or branch being considered.
- The module source location, if available, from `composer.json`, `composer.lock`, or Drupal.org project metadata.
- The custom code scope: `web/modules/custom` and `web/themes/custom`.
- Any local patches applied via `cweagans/composer-patches`, including the patch file path, patch URL, package target, and whether the patch is currently enabled in root `composer.json`.

**Tool guidance for prerequisite resolution:**

- Use `grep_search` with `isRegexp: true` on `composer.lock` to extract the `"name"`, `"version"`, and `"source"` blocks for the package.
- Use `grep_search` on root `composer.json` to find the `"patches"` key under `"extra"`.
- Use `run_in_terminal` to run `lando composer show drupal/<module>` for installed version confirmation.

If the source repository URL is not already known, resolve it before doing diff analysis. Prefer authoritative metadata in this order:

1. `composer.lock` `source.url` and `source.reference`.
2. `composer.json` `repository`, `support.source`, or `source` metadata.
3. Drupal.org project page and release metadata.
4. The upstream VCS repository for the module if it is mirrored elsewhere.

## Phase 0: Upgrade Triage (Always Run First)

Before committing to the full analysis, determine the scope of work.

1. **Classify the version bump** using semantic versioning:
   - **Patch bump** (e.g., `2.1.3 → 2.1.5`): Run patch overlay check and changelog scan only. Skip Phases 2–3 unless patches or known issues are found.
   - **Minor bump** (e.g., `2.1.x → 2.3.x`): Run Phases 1–2, and a targeted Phase 3 focused only on hook, service, and plugin API changes.
   - **Major bump** (e.g., `1.x → 2.x`) or **cross-Drupal-major bump**: Run all phases in full. Flag as highest risk.
   - **Non-SemVer or dev branches**: Treat as a major bump risk by default.

2. **Announce the scope** to the user before proceeding: state which phases will be run and why, so they can redirect if needed.

3. **Create the report stub immediately.** As soon as the target file path is known, create `.copilot_workspace/contrib-upgrade-audit-{module_name}-{from_version}-to-{to_version}.md` using `create_file` with skeleton section headings and an `[IN PROGRESS]` status line at the top. Do not wait until analysis is complete. Append findings to each section incrementally as each phase completes. If `.copilot_workspace/` does not exist, create it first.

   > **This step is mandatory.** If the stub file cannot be created, stop and report the error before proceeding with any analysis.

## Workflow Phase 1: Metadata & Dependency Alignment

> Steps 1, 2, 3, and 4 are independent. Run them in parallel where the host environment supports it.

1. **Issue Queue Check:** Use the `drupal-issue-queue` skill to check the module's issue queue for known breaking changes, upgrade notes, and patch discussions between the current and target versions.
2. **Dry-run dependency resolution:** Use `run_in_terminal` to run `lando composer require "package/name:target_version" --dry-run` and capture any transitive dependency changes or version conflicts.
3. **Version Comparison:** Fetch the NEW version of the contrib module's `*.info.yml` and `composer.json` from the upstream source. Use `fetch_webpage` if needed. Identify changes to:
   - `core_version_requirement`
   - `dependencies` array
   - Required PHP library version constraints
4. **Custom Metadata Audit:**
   - Use `file_search` with pattern `web/modules/custom/**/*.info.yml` and `grep_search` to find explicit dependencies on the module machine name.
   - Use `file_search` with pattern `web/modules/custom/**/composer.json` and `grep_search` for library version constraints.
   - **Conflict Check:** Flag if the new contrib version requires a library (e.g., `guzzlehttp/guzzle:^7.8`) that a custom module has pinned to an older version (e.g., `^7.5`).
5. **Package Provenance:** Record the exact installed package version, source URL, and source reference from `composer.lock` so later diff analysis can compare against the exact upstream revision.
6. **Patch Inventory:** Read root `composer.json` `extra.patches` and any local patch files identified by `file_search` to find all patches applied to the target package or its dependencies. For each patch, record:
   - package target
   - patch title or description
   - patch source URL or local file path
   - whether it is likely already upstreamed
   - whether it alters the same files or symbols being upgraded
7. **Config Schema Audit:** Fetch the new version's `*.schema.yml` files using `fetch_webpage` or by inspecting the upstream diff. Compare against the installed version's schema. Flag:
   - Removed config keys (will break existing exported config in `config/sync/` on import).
   - Renamed keys or changed value types.
   - New required keys without default values.
8. **Update Hook Detection:** Inspect the new version's `*.install` file for `hook_update_N()` and `hook_post_update_NAME()` implementations added since the installed version. For each found:
   - Summarize what the update hook does (data migration, schema change, config restructure).
   - Flag whether it has a dependency on other modules or config that may fail on this site.
   - Note that `lando drush updatedb` must be run post-upgrade and its effects are irreversible without a database restore.

## Workflow Phase 2: Ghost Dependency & Surface Analysis

Identify "Ghost Dependencies"—custom code that uses the module but doesn't declare it in metadata.

> Steps 1, 2, 3, and 4 are independent. Run them in parallel where the host environment supports it.

1. **Namespace Scan:** Identify the PHP namespace of the contrib module (e.g., `Drupal\webform\`). Use `grep_search` with `isRegexp: true` to search `web/modules/custom` and `web/themes/custom` for `use` statements or FQCNs matching this namespace.
2. **Service ID Search:** Parse the contrib module's `*.services.yml` to extract service IDs. Use `grep_search` to find:
   - `\Drupal::service('service_id')` calls in `.php`, `.module`, and `.theme` files.
   - Service decoration entries in custom `*.services.yml` files.
   - Constructor injection type-hints matching contrib service classes.
3. **Hook & Plugin Audit:** Use `grep_search` to find custom implementations of hooks or plugins defined by the module (e.g., `@WebformHandler`, `hook_module_name_data_alter`, `hook_module_name_form_alter`).
4. **Test Coverage Inventory:** Identify existing test coverage that exercises the module being upgraded:
   - Use `grep_search` on `features/` for Behat scenarios referencing the module name, its routes, or its page text.
   - Use `grep_search` on `tests/` for PHPUnit test files referencing the module's namespace or service IDs.
   - Record these files in the handoff report — they form the regression test suite for this upgrade and must be re-run after applying the update.
   - If no tests exist for this module's functionality, flag it explicitly as a risk in the report.

## Workflow Phase 3: Breaking Change Detection

1. **Resolve the exact old and new refs:**
   - Read the installed version, source URL, and source reference from `composer.lock`.
   - Resolve the target version to a tag, branch, or commit SHA from Composer or release metadata.
   - Prefer a compare route when the upstream repository exposes one.
2. **Choose the best diff source:**
   - GitHub upstream: use `https://github.com/org/repo/compare/old_ref...new_ref`
   - GitLab upstream: use `https://gitlab.com/org/repo/-/compare/old_ref...new_ref`
   - Drupal.org git: use the project repository compare route for the relevant tag or commit range.
   - No compare route available: diff the unpacked release archives for the old and new versions.
   - Use `fetch_webpage` to retrieve the compare diff or release page content.
3. **Diff Logic:** When reviewing the diff, look for:
   - Changes to `public` or `protected` method signatures.
   - Removal or renaming of classes or interfaces.
   - Changes to service constructor arguments, service IDs, or tags that break decorators or service subscribers.
   - New or removed hooks, plugin derivatives, events, or alter hooks.
   - Changes to `*.schema.yml` (cross-reference findings from Phase 1 Step 7).
   - Behavioral changes in defaults, validation, return types, or cacheability metadata.
4. **Patch Overlay Check:** Compare each applied patch against the upstream diff and determine whether it is:
   - still required for the target version,
   - already included upstream,
   - now conflicting with upstream changes, or
   - touching custom integration points that need retesting.
5. **High-Signal Evidence:** Record the compare URL, tag names, commit SHAs, release archive names, and patch file paths used to generate the diff so the final report can be audited later.

## Output Requirements

The report file is created at the end of Phase 0 (Step 3) and filled incrementally. The target path is:

```
.copilot_workspace/contrib-upgrade-audit-{module_name}-{from_version}-to-{to_version}.md
```

Do not overwrite an existing report — append a timestamp suffix (e.g., `-20260329`) if the target filename already exists.

Always output an "Upgrade Impact Assessment" with the following sections:

---

### Summary

State the upgrade scope (patch/minor/major), the phases run, and the top 3–5 findings a developer needs to know before touching the codebase. Include an overall **Upgrade Complexity Estimate**:

| Rating | Meaning |
| :--- | :--- |
| **Low** | Patch bump, no breaking API changes, patches still apply, no config schema changes. Safe to apply with standard `drush updatedb` and a smoke test. |
| **Medium** | Minor API changes, one or more patches need review, config schema has additive changes. Requires targeted code changes and regression testing. |
| **High** | Major API changes, breaking service/hook/class removals, config schema restructure, multiple patches conflict. Requires a developer sprint and a full regression suite. |

---

### 1. Metadata Conflicts

| File | Conflict Type | Impact | Complexity |
| :--- | :--- | :--- | :--- |
| `custom_mod.info.yml` | Missing Dependency | **High** | Low — add one line to `.info.yml` |
| `root/composer.json` | Version Constraint | **Medium** | Medium — requires composer constraint resolution |

---

### 2. Config Schema & Update Hook Risk

| Item | Type | Risk | Notes |
| :--- | :--- | :--- | :--- |
| `module.settings` key `legacy_key` removed | Schema Change | **High** | Existing config in `config/sync/` will fail `config:import` |
| `hook_update_9001()` restructures field data | Update Hook | **High** | Data migration; irreversible without a DB restore |

---

### 3. Code Impact (The "X-Ray")

| Custom File | Symbol Used | Change Type | Action Required | Complexity |
| :--- | :--- | :--- | :--- | :--- |
| `CustomAuth.php` | `WebformSubmission` | Method Signature | Update `save()` arguments | Low |
| `services.yml` | `service.decorator` | Constructor Change | Update `__construct` in custom decorator | Medium |

---

### 4. Ghost Dependencies Found

List any files using classes or services from the module that are **NOT** listed in the custom module's `.info.yml` or `composer.json`. For each entry, note whether a dependency declaration should be added or whether the usage should be refactored away.

---

### 5. Test Coverage Inventory

| Test File | Type | Covers | Notes |
| :--- | :--- | :--- | :--- |
| `features/webform-submission.feature` | Behat | Submission flow | Must re-run post-upgrade |
| `tests/src/Functional/WebformTest.php` | PHPUnit Functional | Form rendering | Must re-run post-upgrade |

If no tests exist for the upgraded module's functionality, state this explicitly and flag it as a risk.

---

### 6. Diff Evidence

List the exact upstream source used for comparison:

- Installed package version and source reference.
- Target version and source reference.
- Compare URL or archive filenames.
- Release notes or issue queue references that explain the change.

---

### 7. Patch Overlay

| Patch | Package | Still Required? | Upstreamed? | Conflicts? |
| :--- | :--- | :--- | :--- | :--- |
| `patches/webform-fix.patch` | `drupal/webform` | Yes | No | No |

---

### 8. Developer Handoff References

Include the artifacts below so a developer agent can perform solution design and refactoring without re-discovering the same context:

- The exact package name, installed version, target version, and source reference from `composer.lock`.
- The upstream compare URL, release tag, commit SHA, or release archive names used to analyze the upgrade.
- Any local `cweagans/composer-patches` entries, including patch file paths and patch URLs.
- The issue queue references, release notes, or upstream issue links that explain behavior changes.
- The list of impacted custom files, classes, hooks, services, or plugins, with a short note on why each is affected.
- The relevant custom code snippets or symbol names that need redesign, not just the filenames.
- The metadata findings from `*.info.yml` and `composer.json` that affect dependency declarations.
- The test files identified in Phase 2 that form the regression validation suite.
- Any open questions, assumptions, or unknowns that must be resolved before implementation.

---

## Completion Gates

Do not conclude this skill's execution and do not respond to the user with a summary until **all** of the following conditions are verified:

1. **Report file exists on disk and is non-empty.** Use `file_search` to confirm `.copilot_workspace/contrib-upgrade-audit-{module_name}-{from_version}-to-{to_version}.md` exists. If it does not, create it now from the findings gathered so far before proceeding to gate 2.

2. **All required sections are populated.** The report must contain non-placeholder content in: Summary, Section 1 (Metadata Conflicts), Section 2 (Config Schema & Update Hook Risk), Section 3 (Code Impact), Section 4 (Ghost Dependencies), Section 5 (Test Coverage Inventory), Section 6 (Diff Evidence), Section 7 (Patch Overlay), and Section 8 (Developer Handoff References). A section may state "No findings" only if the corresponding analysis phase was actually run and produced no results.

3. **Patch overlay is complete.** Every patch in `extra.patches` targeting this module has been assessed. No patch entry may be left as "Unknown" without a documented reason.

4. **Test coverage inventory is explicit.** Either test files are listed, or the absence of tests is recorded as an explicit risk finding — not omitted.

If any gate fails, complete the missing work before responding.
