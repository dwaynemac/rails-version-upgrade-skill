---
name: rails-version-upgrade
description: >-
  Upgrade a Ruby on Rails application from one version to another (e.g. 6.1 to
  7.0, 7.2 to 8.0, or any sequential path). Covers pre-upgrade assessment,
  dual-boot setup, Ruby version alignment, gem compatibility, sequential
  version hops, framework defaults migration, config file updates, and
  broken-build triage. Use this skill whenever the goal is to move a Rails app
  to a newer Rails version.
---

# Rails Version Upgrade

Upgrade a Ruby on Rails application safely by following a sequential process
derived from the official Rails upgrade guides and FastRuby.io's methodology.

## Core Principles

1. **Never skip versions.** Always compute the path from
   `data/rails_versions.yml` and upgrade one minor version at a time. If the
   target is not in the graph, stop and follow
   `maintainers/add-new-rails-version.md` before proceeding.
2. **Tests first.** A passing test suite before the upgrade is non-negotiable.
   Aim for ≥80% coverage. The suite must stay green after each hop.
3. **Dual boot.** Run the app against both the current and target Rails version
   simultaneously via `next_rails`. This is the single most important technique
   for safe, debuggable upgrades.
4. **Align defaults before bumping versions.** `config.load_defaults` must
   match the *current* Rails version before moving to the next one.
5. **Fix deprecations before upgrading.** Treat every deprecation warning in
   the current version as a hard requirement, not optional hygiene.
6. **Upgrade Ruby and Rails separately.** First reach the minimum Ruby version
   required by the target Rails, then upgrade Rails.
7. **Separate required upgrade work from optional modernization.** Do not migrate
   asset pipeline, jobs, cache, cable, authentication, or deployment tooling as
   part of the Rails bump unless the user explicitly asks for that scope.

## Core Workflow

1. **Assess** – establish test coverage baseline, identify the current and
   target versions, classify the app surfaces, and map the sequential hop path
   from `data/rails_versions.yml`.
2. **Align current defaults** – ensure `config.load_defaults` matches the
   running Rails version; work through `new_framework_defaults_X_Y.rb`.
3. **Fix deprecations** – run the full suite, collect every
   `DEPRECATION WARNING`, resolve them all in the current version.
4. **Set up dual boot** – install `next_rails`, create `Gemfile.next`,
   configure CI to run both gemfiles.
5. **Bump one version** – update the `rails` constraint in `Gemfile.next`,
   run `bundle install` against it, verify the app boots.
6. **Run `bin/rails app:update`** – accept/merge config changes interactively;
   review `new_framework_defaults_X_Y.rb`.
7. **Fix gem compatibility** – resolve incompatible gems one at a time using
   `bundle exec bundle_report compatibility --rails-version=<target>` /
   RailsBump; never mass-update.
8. **Fix the broken build** – attack errors before failures; start with model
   tests; batch fixes by root cause.
9. **Migrate framework defaults** – uncomment defaults in
   `new_framework_defaults_X_Y.rb` one by one, verify the suite after each.
10. **Promote** – once both Gemfiles pass, set `config.load_defaults` to the
    new version, remove dual-boot conditionals, delete `Gemfile.next`.
11. **Repeat** for the next hop until reaching the target version.

## Guardrails

- Never update all gems at once (`bundle update` with no args). Update only
  what the Rails bump requires, then resolve conflicts conservatively.
- Never disable or skip the test suite to "speed up" an upgrade step.
- Do not change `config.load_defaults` and bump the Rails version in the same
  commit — these are two distinct, separable changes.
- When a deprecation warning says "this will be removed in Rails X.Y", treat
  it as already removed.
- Do not attempt to upgrade if the application has no test suite at all —
  block and recommend writing tests (or acceptance criteria) first.
- Dual-boot conditionals (`NextRails.next?`) are intentional temporary debt;
  document them and remove them after promotion.
- Treat Rails 8 defaults/features such as Propshaft, Solid Queue, Solid Cache,
  Solid Cable, the authentication generator, Kamal, and Thruster as optional
  modernization. Keep them out of the framework upgrade PR unless requested.
- Never blindly accept `bin/rails app:update` overwrites. Preserve local
  customizations and categorize each diff before applying it.

## Load References Selectively

Open only the file needed for the current step.

- Version graph and Ruby compatibility gates: `data/rails_versions.yml`
- Step-by-step universal process: `references/upgrade-playbook.md`
- Rails 7.2 → 8.0 focused playbook: `references/rails-7-2-to-8-0.md`
- Breaking changes per version: `references/version-specific-notes.md`
- Gem / dependency management: `references/gem-compatibility.md`
- `config.load_defaults` migration: `references/framework-defaults.md`
- Fixing a red test suite: `references/broken-build-triage.md`
- Smoke script and coverage targets: `references/smoke-and-validation.md`
- Risk matrix (planning / triage): `references/risk-matrix.md`
- Maintainer workflow for future Rails versions: `maintainers/add-new-rails-version.md`

Use `references/INDEX.md` for the full catalog.

## Key External Resources

- Official upgrade guide: https://guides.rubyonrails.org/upgrading_ruby_on_rails.html
- Rails release notes index: https://guides.rubyonrails.org/ (see Release Notes section)
- Ruby/Rails compatibility table: https://www.fastruby.io/blog/ruby/rails/versions/compatibility-table.html
- RailsBump gem compatibility checker: https://railsbump.org/
- RailsDiff config diff tool: https://railsdiff.org/
- `next_rails` gem: https://github.com/fastruby/next_rails
- FastRuby.io upgrade series: https://www.fastruby.io/blog/rails/upgrade/rails-upgrade-series.html
- FastRuby.io open-source upgrade skill: https://github.com/ombulabs/claude-code_rails-upgrade-skill
