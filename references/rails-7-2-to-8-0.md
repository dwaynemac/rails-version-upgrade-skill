# Rails 7.2 to 8.0 Focused Playbook

Use this reference when the current app is already on Rails 7.2 and the target
is Rails 8.0. This is a single-hop upgrade, but it still needs Ruby gating,
dependency review, `app:update`, framework-default staging, and validation.

Primary sources:
- Official upgrade guide: https://guides.rubyonrails.org/upgrading_ruby_on_rails.html#upgrading-from-rails-7-2-to-rails-8-0
- Rails 8.0 release notes: https://guides.rubyonrails.org/8_0_release_notes.html
- FastRuby.io 7.2 to 8.0 guide: https://www.fastruby.io/blog/upgrade-rails-7-2-to-8-0.html
- RailsDiff: https://railsdiff.org/7.2.0/8.0.0
- RailsBump: https://railsbump.org/

---

## Scope

Required upgrade work:
- Move from the latest Rails 7.2 patch to the latest Rails 8.0 patch.
- Upgrade Ruby to 3.2.0 or newer before the Rails bump if needed.
- Resolve removed Rails 7.x APIs that are gone in Rails 8.0.
- Run `bin/rails app:update` and review generated diffs.
- Keep `config.load_defaults 7.2` until the framework-only upgrade is stable.
- Validate boot, tests, assets, jobs, database bootstrap, and production-like
  behavior before promotion.

Out of scope unless explicitly requested:
- Sprockets to Propshaft migration.
- Existing job backend to Solid Queue migration.
- Existing cache backend to Solid Cache migration.
- Existing Action Cable adapter to Solid Cable migration.
- Existing authentication system to the Rails authentication generator.
- Existing deployment stack to Kamal 2 or Thruster.

Rails 8.0 makes those technologies defaults or first-class options for new
applications. Existing applications do not need to adopt them to complete the
framework upgrade.

---

## Preflight Gates

### 1. Start from the latest Rails 7.2 patch

```ruby
# Gemfile
gem "rails", "~> 7.2.0"
```

```bash
bundle update rails
bin/rails test   # or bundle exec rspec
```

Fix deprecations on Rails 7.2 before introducing Rails 8.0. Removed APIs in
Rails 8.0 should already have warned in 7.x.

### 2. Upgrade Ruby separately

Rails 8.0 requires Ruby 3.2.0 or newer.

```bash
ruby -v
bundle platform --ruby
```

If the app is below Ruby 3.2, upgrade Ruby in its own branch/PR first and get
the Rails 7.2 test suite green before touching the Rails version.

### 3. Inventory high-risk Rails 8.0 removals

```bash
git grep -nE 'read_encrypted_secrets|rails/console/(app|helpers)|Rails::ConsoleMethods|allow_deprecated_parameters_hash_equality|form_with\([^\n]*model:\s*nil|tag\.[a-z_]+\([^\n]*do|commit_transaction_on_non_local_return|allow_deprecated_singular_associations_name|warn_on_records_fetched_greater_than|sqlite3_deprecated_warning|ConnectionPool#connection|\.connection\b|cache_dump_filename|SCHEMA_CACHE|ActiveSupport::ProxyObject|attr_internal_naming_format|use_big_decimal_serializer|enqueue_after_transaction_commit|enum\s+[a-zA-Z_]+:' -- '*.rb' '*.erb' '*.haml' '*.slim'
```

Treat matches as a review queue, not automatic bugs. Some patterns are broad
and need manual confirmation.

### 4. Check dependency compatibility

```bash
bundle outdated || true
bundle exec bundle_report compatibility --rails-version=8.0.0
```

Use the actual target patch instead of `8.0.0` and ensure `next_rails` is
available in the selected bundle.

Also check the app's `Gemfile.lock` with RailsBump for Rails 8.0 compatibility.
Update only gems required by the Rails/Ruby bump or confirmed incompatibilities.
Do not run `bundle update` with no arguments.

---

## Dual-Boot Setup

Prefer dual boot for product apps so current work can continue on Rails 7.2
while CI proves Rails 8.0 compatibility.

### Option A: `next_rails`

```bash
gem install next_rails
next --init
```

```ruby
# Gemfile
if NextRails.next?
  gem "rails", "~> 8.0.0"
else
  gem "rails", "~> 7.2.0"
end
```

### Option B: `eval_gemfile`

```ruby
# Gemfile.next
eval_gemfile "Gemfile"

gem "rails", "~> 8.0.0"
# Add temporary compatibility pins here only when necessary.
```

Install both variants:

```bash
bundle install
BUNDLE_GEMFILE=Gemfile.next bundle install
```

Verify the target app boots:

```bash
BUNDLE_GEMFILE=Gemfile.next bundle exec rails runner 'puts Rails.version'
```

---

## Version Bump and App Update

### 1. Update Rails

```bash
BUNDLE_GEMFILE=Gemfile.next bundle update rails
```

If using Rails JavaScript packages with `jsbundling-rails`, align package
versions as part of the same hop:

```bash
BUNDLE_GEMFILE=Gemfile.next bundle exec rails javascript:install
```

### 2. Run `app:update`

```bash
THOR_DIFF="git diff --no-index --color" \
  BUNDLE_GEMFILE=Gemfile.next bundle exec rails app:update
```

Review every generated diff. In particular:
- Keep local changes in `config/application.rb` and environment files.
- Add `config/initializers/new_framework_defaults_8_0.rb` if generated.
- Do not flip `config.load_defaults 8.0` in the same commit as the framework
  bump.
- Do not accept optional Rails 8 stack migrations just because new app templates
  include them.

Use RailsDiff (`https://railsdiff.org/7.2.0/8.0.0`) as a comparison aid, not as
an instruction to overwrite app-specific configuration.

---

## Rails 8.0 Removed APIs Checklist

### Railties

- [ ] Remove `config.read_encrypted_secrets`.
- [ ] Remove requires for `rails/console/app` and `rails/console/helpers`.
- [ ] Replace any `Rails::ConsoleMethods` console extension hook.
- [ ] Replace custom `STATS_DIRECTORIES` mutation with
      `Rails::CodeStatistics.register_directory` if used.
- [ ] Prefer `bin/rails stats` over deprecated `bin/rake stats`.

### Action Pack

- [ ] Remove `config.action_controller.allow_deprecated_parameters_hash_equality`.
- [ ] Replace fragile `params == {}` / `params == hash` checks with explicit
      conversions such as `params.to_unsafe_h` or permitted hashes where
      appropriate.
- [ ] Audit routes drawn with multiple paths if deprecation warnings mention
      routing performance.

### Action View

- [ ] Replace `form_with(model: nil)` with an explicit `url:` or concrete model.
- [ ] Remove content blocks passed to void tag helpers such as `br`, `img`,
      `input`, `meta`, and `link`.

### Active Record

- [ ] Remove `config.active_record.commit_transaction_on_non_local_return`.
- [ ] Remove `config.active_record.allow_deprecated_singular_associations_name`.
- [ ] Remove `config.active_record.warn_on_records_fetched_greater_than`.
- [ ] Remove `config.active_record.sqlite3_deprecated_warning`.
- [ ] Replace unregistered database adapter lookup with a registered adapter.
- [ ] Replace keyword-style enums:

```ruby
# Before
enum status: { active: 0, archived: 1 }, _default: :active

# After
enum :status, { active: 0, archived: 1 }, default: :active
```

- [ ] Replace `ActiveRecord::ConnectionAdapters::ConnectionPool#connection`
      usage with `with_connection` or `checkout`/`checkin` patterns.
- [ ] Remove database-name arguments to `cache_dump_filename` if used.
- [ ] Stop setting `ENV["SCHEMA_CACHE"]`; use current schema cache config.
- [ ] Update fresh-database scripts that expect `db:migrate` to replay every
      migration. Rails 8.0 loads the schema first on a fresh database, then runs
      pending migrations. Use `db:migrate:reset` when full replay is required.

### Active Support

- [ ] Replace `ActiveSupport::ProxyObject` inheritance.
- [ ] Remove `attr_internal_naming_format` values that include an `@` prefix.
- [ ] Pass strings, not arrays of strings, to `ActiveSupport::Deprecation#warn`.
- [ ] Audit slow regular expressions. Rails 8.0 sets `Regexp.timeout = 1` by
      default.

### Active Job

- [ ] Remove `config.active_job.use_big_decimal_serializer`.
- [ ] Replace or consciously defer `enqueue_after_transaction_commit`, which is
      deprecated in Rails 8.0.
- [ ] Verify jobs enqueued inside database transactions still run after commit
      as intended.

### Active Storage

- [ ] If using Azure storage, note that the Azure backend is deprecated in Rails
      8.0. Plan a separate migration; do not mix it into the framework bump
      unless the app is already blocked by it.

---

## New Rails 8.0 Defaults to Stage Separately

Keep these in `new_framework_defaults_8_0.rb` or equivalent staging until the
Rails 8.0 framework-only upgrade is stable.

### `Regexp.timeout = 1`

Risk: long-running regexes can now raise timeout errors. This is a security
improvement against ReDoS, but custom import/parsing/search code may need
testing.

Validation:
- Run request specs and jobs that parse user-provided text.
- Test any CSV, log parsing, search, validation, or sanitizer code with large
  inputs.

### `config.action_dispatch.strict_freshness = true`

Risk: HTTP freshness checks involving `ETag` and `Last-Modified` become stricter.
Apps with custom caching headers may see changed `304 Not Modified` behavior.

Validation:
- Test endpoints that set explicit `ETag`, `Last-Modified`, `fresh_when`, or
  `stale?` headers.
- Verify CDN/proxy behavior in staging if the app relies on conditional GETs.

---

## Optional Rails 8 Features: Defer by Default

| Feature | Rails 8 role | Upgrade guidance |
|---------|--------------|------------------|
| Propshaft | Default asset pipeline for new apps | Keep Sprockets if it is working. Migrate later as its own asset-pipeline project. |
| Solid Queue | Default Active Job backend for new apps | Keep Sidekiq/Resque/Delayed Job/etc. during the framework bump unless the user requests a backend migration. |
| Solid Cache | DB-backed cache option | Keep Redis/Memcached during the framework bump. Move cache store separately if desired. |
| Solid Cable | DB-backed Action Cable adapter | Keep the current shared adapter during the framework bump. Test multi-process broadcasts either way. |
| Authentication generator | New generator for auth skeletons | Do not replace Devise/custom auth during the Rails upgrade. |
| Kamal 2 / Thruster | New app deployment defaults | Do not change deployment topology as part of the framework bump unless requested. |

---

## Validation Checklist

Run this against `Gemfile.next` before promotion and against the promoted
`Gemfile` after removing dual boot.

```bash
BUNDLE_GEMFILE=Gemfile.next bin/rails db:prepare
BUNDLE_GEMFILE=Gemfile.next bin/rails zeitwerk:check
RAILS_ENV=production SECRET_KEY_BASE_DUMMY=1 \
  BUNDLE_GEMFILE=Gemfile.next bin/rails assets:precompile
```

- [ ] App boots in development, test, and production-like configuration.
- [ ] Full Minitest and/or RSpec suite passes.
- [ ] `db:prepare` works on a fresh checkout.
- [ ] `db:migrate` fresh-database behavior is acceptable for CI and deployment.
- [ ] Production asset precompile succeeds.
- [ ] Authentication, sessions, password reset, and signed/encrypted values work.
- [ ] Critical request/API flows pass manually or via system/request tests.
- [ ] Background jobs that enqueue inside transactions are verified end-to-end.
- [ ] Persisted background jobs from the old bundle still deserialize and run.
- [ ] If Action Cable is used, broadcasts work across more than one process.
- [ ] If Active Storage is used, upload, download, variants, and previews work.
- [ ] Cache-backed paths do not store or load framework internals such as
      `ActiveRecord::Relation` objects.
- [ ] Brakeman runs clean enough for the app's CI policy.
- [ ] Staging/canary deploy succeeds before `config.load_defaults 8.0` is flipped.
- [ ] Production monitoring is checked after deploy for incidents scoped to the
      new revision.

### Lockfile and runtime sanity checks

After the Rails 8.0 bundle resolves, verify that the lockfile did not keep
stale Rack 2-era dependencies through transitive constraints.

```bash
bundle exec rails runner 'puts "Rails #{Rails.version}"; puts "Rack #{Rack.release}"; puts "Bundler gemfile=#{Bundler.default_gemfile}"'
bundle exec ruby -e 'puts Bundler.locked_gems.specs.values_at("rails", "rack", "rack-session", "rackup", "rack-cors").compact.map { |s| "#{s.name} #{s.version}" }'
```

Expected shape for a typical Rails 8.0/Rack 3 app:

- `rails 8.0.x`
- `rack 3.2.x`
- `rack-session 2.1.x`
- `rackup 2.3.x`
- `rack-cors 3.0.x` if the app uses CORS middleware

If the main lockfile still contains Rack 2-era versions after promotion, run a
targeted update of the Rack family and any direct gems constraining it. Do not
run an unrestricted `bundle update`.

```bash
bundle update rails rack rack-session rackup rack-cors
```

### Persisted data compatibility checks

Rails upgrades do not only execute new code; they also read data written by the
old bundle. Validate persisted objects that include framework metadata.

- Background jobs: inspect serialized Active Job payloads for `locale`,
  `timezone`, GlobalID arguments, and job class names. Rails restores locale and
  timezone before `perform`, so invalid values such as `en-US` or legacy
  timezone aliases can fail before app code runs. Normalize regional locales to
  app-supported locales (`en-US -> en`, `pt-BR -> pt`) or discard safe no-op jobs
  deliberately.
- Cache entries: do not cache `ActiveRecord::Relation` objects or other Rails
  internals across the upgrade. Cache primitive IDs or plain serialized values,
  then re-query under the new bundle.
- Webhooks: filter non-actionable events at the HTTP boundary when safe, instead
  of enqueueing jobs that immediately no-op. This reduces retry noise when
  serialized job metadata is incompatible.

---

## Common 7.2 to 8.0 Failure Patterns

| Symptom | Likely cause | First move |
|---------|--------------|------------|
| Boot fails on `read_encrypted_secrets` | Removed Railties config | Remove the config and use credentials/secrets management supported by the app |
| Tests fail around params equality | Deprecated parameters-hash equality removed | Convert params explicitly before comparison or assert on permitted values |
| View raises on `form_with(model: nil)` | Removed Action View fallback | Pass `url:` or a real model explicitly |
| View helper raises for `tag.img`/`tag.br` with content | Void tag content support removed | Remove content/block from void tag helper calls |
| Models fail at boot around `enum` | Keyword-argument enum syntax removed | Convert to `enum :name, { ... }, option: value` |
| Code fails on connection pool `connection` | Pool `#connection` removed | Use `with_connection` for scoped access |
| Fresh CI database no longer replays all migrations | Rails 8 schema-first `db:migrate` behavior | Use `db:prepare` normally; use `db:migrate:reset` only when full replay is required |
| Regex-heavy code raises timeout | `Regexp.timeout = 1` default | Optimize regex, bound input size, or set a narrower timeout policy deliberately |
| Jobs behave differently around transactions | Rails 7.2/8.0 transaction enqueue semantics | Make adapter explicit in tests and move side effects to after-commit semantics |
| Background job fails before `perform` with `I18n::InvalidLocale` | Persisted Active Job payload has a regional or unsupported locale | Normalize job locales globally or repair/discard the affected queued jobs |
| Background job fails before `perform` with an invalid timezone | Persisted Active Job payload has a legacy timezone alias | Normalize job timezones during Active Job serialization/deserialization |
| Production cache raises deserialization or connection-pool errors | Cached relation/framework object written by old bundle | Stop caching relations; cache primitive IDs or plain values and re-query |
| ActiveAdmin resource rejects permitted params after upgrade | Nested permit list accidentally passed as a single array | Splat/flatten permit lists deliberately; avoid `<<` when concatenation is intended |
| `LoadError` mentions `active_support/proxy_object` or `active_support/basic_object` | Older gem requires ActiveSupport files removed in Rails 8 | Update the gem; `jbuilder` older than the Rails 8-compatible line is a common cause |
| Groupdate `group_by_week` fails or `.top` is missing | Old Groupdate breaks on ActiveRecord 8; newer Groupdate removed old convenience APIs | Update Groupdate and replace `.top(column, n)` with explicit `group/order/limit/count` |

---

## Promotion

Promote only after the Rails 8.0 dual-boot suite is green.

1. Move the Rails 8.0 constraint from `Gemfile.next` to `Gemfile`.
2. Run targeted `bundle update` commands as needed to reconcile `Gemfile.lock`.
3. Verify the promoted main lockfile has Rails 8 and Rack 3-compatible versions.
4. Remove temporary `NextRails.next?` branches and `Gemfile.next` files.
5. Remove deployment build args or environment variables that still point at
   `Gemfile.next`.
6. Keep `config.load_defaults 7.2` until the framework-only upgrade has been
   stable in staging/canary.
7. Enable Rails 8.0 defaults one at a time in a follow-up defaults PR.
8. Remove temporary compatibility pins and shims once production is stable.

Before the first production deploy without dual boot, check both the repository
and the deployment platform for stale `BUNDLE_GEMFILE` configuration.

```bash
git grep -nE 'BUNDLE_GEMFILE|Gemfile.next' -- . ':!.git'
env | grep BUNDLE_GEMFILE
```

Bundler always respects `ENV["BUNDLE_GEMFILE"]`. If the platform still injects
`BUNDLE_GEMFILE=/app/Gemfile.next` or `/rails/Gemfile.next` after the file is
deleted, migrations and workers will fail with `Bundler::GemfileNotFound` even
though the Dockerfile or app boot code defaults to `Gemfile`.

After deploy, inspect production monitoring by revision. Look specifically for:

- delayed jobs retried from before the promotion
- cache deserialization failures
- malformed Rack 3 request parsing errors
- webhook jobs that can be safely discarded or filtered earlier
