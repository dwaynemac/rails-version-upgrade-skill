# Gem Compatibility During a Rails Upgrade

Managing third-party gem dependencies is usually the most time-consuming part
of a Rails upgrade. This document covers the systematic approach to diagnosing
and resolving incompatibilities without destabilising the rest of the app.

---

## Guiding Principle: One Gem at a Time

Never run `bundle update` with no arguments during an upgrade. Doing so
resolves all constraints from scratch and can introduce dozens of silent
behaviour changes at once, making it impossible to trace regressions.

Always update gems in targeted, minimal increments.

---

## Step 1 — Generate an Incompatibility Report

### Using `next_rails`

```bash
# Against the target Gemfile
BUNDLE_GEMFILE=Gemfile.next bundle exec next_rails bundle_report
```

Produces a list of gems that:
- Have no version compatible with the target Rails
- Have compatible versions but require a version bump

### Using RailsBump

Paste the contents of `Gemfile.lock` into https://railsbump.org/ and select
the target Rails version. The tool shows which gems are compatible,
which need updating, and which have no compatible release.

### Manual approach

```bash
BUNDLE_GEMFILE=Gemfile.next bundle install 2>&1 | grep -A 3 "Bundler could not find"
```

---

## Step 2 — Categorise Each Incompatible Gem

For each gem that is not compatible:

| Category | Description | Action |
|----------|-------------|--------|
| **Compatible version available** | A newer release supports the target Rails | Bump the version constraint |
| **Needs conditional** | API changed between Rails versions | Use `NextRails.next?` or `rails_version >= X` |
| **Fork / patch needed** | Unmaintained gem with no compatible release | Fork, vendor, or replace |
| **Already replaceable** | Gem duplicates Rails functionality added in the new version | Remove and use built-in |

---

## Step 3 — Update One Gem at a Time

```bash
# Update a single gem (resolves its dependency tree minimally)
BUNDLE_GEMFILE=Gemfile.next bundle update <gem_name>

# Run the test suite after each update
BUNDLE_GEMFILE=Gemfile.next bundle exec rails test
```

If a gem update breaks tests, address the breakage before moving to the next
gem. Stacking unverified updates makes root-cause analysis much harder.

### Review the complete lockfile diff

Start from the baseline lockfile (copy it to `Gemfile.next.lock` when creating
that bundle) and prefer a targeted conservative update:

```bash
BUNDLE_GEMFILE=Gemfile.next bundle update rails --conservative
```

[Bundler's update documentation](https://bundler.io/man/bundle-update.1.html)
explains conservative resolution. It does not guarantee that only Rails gems
change: required dependency changes and other selected transitive versions
still need review. Compare every added, removed, or changed gem with the
baseline and record why each change is needed.

If an unrelated gem moves unnecessarily, temporarily constrain it to its
baseline version in the target bundle and resolve again. Use a conditional
requirement for an existing direct dependency instead of declaring it twice.
Do not retain an incompatible or vulnerable baseline version merely to reduce
the diff. Remove temporary resolution pins before promotion, run `bundle
install`, inspect the resulting lockfile again, and validate that final bundle.
Do not hand-edit dependency edges in the lockfile or claim tests from an earlier
resolution prove the final one.

---

## Step 4 — Writing Dual-Boot Compatible Code

When a gem changes its API between the Rails versions being dual-booted,
write version-conditional code using `NextRails.next?`:

```ruby
if NextRails.next?
  # Code compatible with the target Rails / target gem version
  Foo.new_api(bar)
else
  # Code compatible with the current Rails / current gem version
  Foo.old_api(bar)
end
```

In views, you can check the gem version directly:

```ruby
if Gem::Version.new(SomeGem::VERSION) >= Gem::Version.new("2.0")
  # new behaviour
else
  # old behaviour
end
```

**Remove all such conditionals after promotion** (Phase 9 of the playbook).
Search for `NextRails.next?` and `Gemfile.next` references once the upgrade
is complete.

---

## Step 5 — Dealing with Unmaintained Gems

If a gem has no release compatible with the target Rails, evaluate in order:

1. **Check open PRs / issues** — a fix may already exist in a branch or fork.
2. **Apply a monkey-patch** behind `NextRails.next?` temporarily.
3. **Fork the gem** and publish a compatibility release. Prefer pointing
   Gemfile at the fork's git ref during the upgrade:

```ruby
if NextRails.next?
  gem 'problematic_gem', github: 'your-fork/problematic_gem', branch: 'rails-8-compat'
else
  gem 'problematic_gem', '~> 1.4'
end
```

4. **Replace the gem** with a maintained alternative or Rails built-in.

---

## Common Gem Categories and Migration Patterns

### Authentication gems

| Gem | Rails 8 status | Notes |
|-----|---------------|-------|
| `devise` | Verify lazy-route and session behavior on Rails 8 | Devise 5 includes lazy-route support; see the compatibility note below |
| `clearance` | Compatible (≥ 2.x) | |
| `authlogic` | Unmaintained | Consider migrating to `devise` or Rails 8's built-in generator |
| `sorcery` | Compatible (≥ 1.x) | |

#### Devise: lazy routes and token continuity

Rails 8 lazily loads routes in development and test. Devise 4 can access
mappings before those routes are loaded; symptoms include missing mappings or
order-dependent session deserialization failures. In one reproduced upgrade,
a temporary test `RouteSet` triggered Warden configuration before application
mappings existed, and a restored user became a Hash. The same seed passed on
Rails 7.2 and failed repeatedly on Rails 8 with Devise 4.9.3; Devise 5.0.4 passed.

[Devise's upstream fix](https://github.com/heartcombo/devise/pull/5728) loads
routes before accessing mappings, and the
[Devise 5 changelog](https://github.com/heartcombo/devise/blob/v5.0.4/CHANGELOG.md)
documents Rails 8 support. Reproduce the app's failure before selecting a
compatible release; these example versions are not universal pins. Verify the
failing seed, full suite, login, and session restoration after the change.

A Devise major update also changes default secret discovery. Preserve an
explicit `Devise.secret_key` and verify that the effective signing key remains
the same without printing it. Previously issued password-reset, confirmation,
and unlock tokens can become invalid if the fallback key changes. Review the
major-version changelog and test token continuity before promotion.

### Background jobs

| Gem | Notes |
|-----|-------|
| `sidekiq` | Generally compatible; check major versions |
| `resque` | Compatible with adapter gem |
| `delayed_job` | Compatible |
| `solid_queue` | Rails 8 built-in alternative; no separate gem needed for new apps |

### Asset pipeline

| Gem | Notes |
|-----|-------|
| `sprockets-rails` | No longer auto-included in Rails 7+; add explicitly if needed |
| `sass-rails` | Use `dartsass-rails` or `cssbundling-rails` for Rails 7+ |
| `webpacker` | Deprecated in Rails 7; replace with `jsbundling-rails` + esbuild/rollup/webpack |
| `propshaft` | Default in new Rails 8 apps; opt-in for existing apps |

### Testing

| Gem | Notes |
|-----|-------|
| `factory_bot_rails` | Generally compatible; ensure ≥ 6.x |
| `capybara` | Generally compatible; ensure ≥ 3.x |
| `brakeman` | Recommended CI gate; Rails 7.2 made it a default for new apps |
| `webmock` | Generally compatible |
| `vcr` | Generally compatible |

### Pagination

| Gem | Notes |
|-----|-------|
| `kaminari` | Compatible (≥ 1.x) |
| `will_paginate` | Compatible |
| `pagy` | Compatible; no Rails coupling |

### File uploads

| Gem | Notes |
|-----|-------|
| `carrierwave` | Compatible (≥ 2.x); consider migrating to Active Storage |
| `paperclip` | Unmaintained since 2018; must migrate to Active Storage or CarrierWave |
| `shrine` | Compatible |

### Other common gems

| Gem | Notes |
|-----|-------|
| `cancancan` | Compatible (≥ 3.x) |
| `pundit` | Compatible |
| `ransack` | Compatible; check major version vs Rails version matrix |
| `aasm` | Compatible (≥ 5.x) |
| `scenic` | Compatible |
| `acts-as-taggable-on` | Rails 8 requires a compatible major; `12.x` supports Rails 8.0 |
| `pg` | Compatible (≥ 1.x) |
| `redis` | Compatible; `redis` v5 changed the API — audit usages |
| `puma` | Compatible (≥ 5.x); bridge family `7.2.x`, newest `8.0.x`; set thread counts explicitly |

### Rails 8.0 notes from production upgrades

| Gem | Rails 8 symptom | Action |
|-----|-----------------|--------|
| `jbuilder` | Boot fails with `LoadError` for `active_support/proxy_object` or `active_support/basic_object` | Update to a Rails 8-compatible release such as `2.15.x` |
| `groupdate` | `group_by_week` can fail on ActiveRecord 8; old `.top(column, limit)` convenience usage may be unavailable after updating | Update to a Rails 8-compatible release such as `6.7.x`; replace `.top` with explicit ActiveRecord grouping |
| `active_admin_import` | Older releases can constrain ActiveAdmin/import dependencies below Rails 8-compatible versions | Update to a Rails 8-compatible major; verify import flows in ActiveAdmin request specs |
| `active_storage_validations` | Older releases may pull stale Active Storage assumptions | Update with Rails/Active Storage when the lockfile is promoted |
| `delayed_job` / `delayed_job_active_record` | Old releases constrained ActiveSupport/ActiveRecord below Rails 8 | Use releases that allow Rails 8 and verify persisted queued jobs deserialize |
| `rack`, `rack-session`, `rackup`, `rack-cors` | Rails 8 does not itself imply Rack 3; server and middleware constraints can differ | Check exact release gemspecs and application compatibility before updating; see the focused Rails 7.2 → 8.0 playbook |

When replacing Groupdate's old `.top` helper, preserve the existing return shape
with plain ActiveRecord:

```ruby
relation.group(column).reorder(Arel.sql("COUNT(*) DESC")).limit(limit).count
```

If no limit was previously used, skip the `limit` call but keep the explicit
`group` and `COUNT(*) DESC` ordering.

---

## Verifying Gem Compatibility After an Upgrade

After all gem conflicts are resolved and the suite is green, run:

```bash
bundle exec bundle-audit check --update
```

to ensure no gems have known security vulnerabilities. Install with:

```bash
gem install bundler-audit
```
