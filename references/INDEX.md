# References Index

## data/rails_versions.yml
Version graph, Ruby compatibility gates, per-hop guide pointers, and optional
modernization categories. Use this before planning an upgrade path.

## maintainers/add-new-rails-version.md
Maintainer workflow for adding Rails 8.2, 9.0, or later support without editing
the main skill every time. Use when the requested target version is not present
in `data/rails_versions.yml`.

## upgrade-playbook.md
Universal step-by-step upgrade process: pre-upgrade assessment, dual-boot
setup, version bump, `app:update`, gem compatibility, broken build, defaults
migration, and promotion. Use this as the primary execution guide for any
Rails version upgrade.

## rails-7-2-to-8-0.md
Focused single-hop playbook for Rails 7.2 to Rails 8.0. Covers Ruby 3.2 gating,
Rails 8.0 removed APIs, `app:update`, Rails 8.0 defaults, optional Rails 8
modernization boundaries, validation, and common failure patterns. Use this
whenever the current app is already on Rails 7.2 and the target is Rails 8.0.

## version-specific-notes.md
Per-version breaking changes, required actions, and new features for each
minor Rails version from 6.0 through 8.1. Includes a template section for
adding support for future Rails versions. Use this when diagnosing failures
or planning the work for a specific hop.

## gem-compatibility.md
How to audit, categorise, and update third-party gems during an upgrade.
Covers `bundle_report compatibility`, RailsBump, one-gem-at-a-time update
strategy, dual-boot conditional code patterns, and a compatibility table for
common gems across Rails versions.

## framework-defaults.md
Explains `config.load_defaults`, how to migrate defaults one at a time using
`new_framework_defaults_X_Y.rb`, risk-tiering of defaults, and special
migration procedures for cache format and cookie serialiser changes.

## broken-build-triage.md
Systematic approach to fixing a red test suite after a version bump. Covers
order of operations (load errors → errors → failures), a quick-reference
symptom→cause→first-move table, dual-boot debugging techniques, common failure
patterns by category, and daily workflow advice.

## smoke-and-validation.md
Smoke bash script (`bin/upgrade-smoke`) that covers `db:prepare`,
`zeitwerk:check`, `assets:precompile`, Minitest/RSpec, and Brakeman in a
single executable gate. Also includes the per-area test coverage target table.
Run after every minor-version hop.

## risk-matrix.md
Nine-row risk matrix covering likelihood, impact, detection, and mitigation for
the most common Rails upgrade risks (Ruby compatibility, Zeitwerk, cookies,
cache serialization, jobs, asset pipeline, schema bootstrap, and hidden
production regressions). Use during planning and when triaging unexpected
failures.
