# Broken Build Triage

After bumping the Rails version in `Gemfile.next` and running the test suite,
you will almost certainly see a red build. This document covers the systematic
approach to turning it green, derived from FastRuby.io's methodology.

---

## Mindset

Fixing a broken build after a version bump is detective work, not a mechanical
checklist. Progress is non-linear: each batch of fixes reveals new failures.
Do not estimate completion by failure count — the last 5% is usually the
hardest 50% of the work.

---

## Quick-Reference: Symptom → Cause → First Move

| Symptom | Likely cause | First move |
|---------|-------------|-----------|
| `NameError: uninitialized constant` during boot | Zeitwerk naming violation, initialization autoloading, `ActiveSupport::Dependencies` usage, manual `require` of app code | Run `bin/rails zeitwerk:check`; grep for `ActiveSupport::Dependencies`, `config.autoloader`, `require_dependency` |
| Users logged out / signed or encrypted values unreadable | Missing cookie rotator or digest migration plan | Restore rotator, redeploy, then phase digest changes properly |
| Cache miss storm or unreadable cache entries after rollout | Cache serializer format changed too early in a rolling deploy | Keep legacy cache format for the first deploy; flip after all nodes are upgraded |
| Tests suddenly execute real jobs | Rails 7.2 now respects configured `queue_adapter` in all tests | Set `config.active_job.queue_adapter = :test` explicitly unless you intend real-adapter tests |
| Controller/integration tests changed exception behavior | Rails 7.1 changed `show_exceptions` semantics | Replace `true/false` with `:all/:rescuable/:none` deliberately |
| Missing assets or JS import errors in production | Sprockets omitted, Propshaft migration incomplete, or `@rails/ujs` import syntax outdated | Precompile in CI; verify asset pipeline choice; update UJS imports |
| Background broadcasts work locally but not across processes | Cable adapter not shared between processes | Verify shared adapter in staging; keep Redis/PostgreSQL or adopt Solid Cable separately |

---

## Establish whether a failure is new

A passing randomized baseline does not prove that every test order passes.
Before changing code for a target-suite failure, replay its exact seed on the
unchanged current-version code and lockfile, with the same test selection,
worker count, environment, and database setup. For example:

```bash
# Use the seed printed by the failing target run.
BUNDLE_GEMFILE=Gemfile.next PARALLEL_WORKERS=1 bundle exec rails test --seed 45784
PARALLEL_WORKERS=1 bundle exec rails test --seed 45784
# RSpec equivalent: bundle exec rspec --seed 45784
```

Keep a clean baseline checkout when application compatibility edits already
exist; merely switching Gemfiles would otherwise test modified code. If the
same failure occurs on the baseline, record it as pre-existing and stop the
upgrade until the baseline issue is resolved separately. Record target-only
failures separately; a shared failure does not classify the whole run.

Preserve the failing full-suite order when an isolated example passes. For
example, an earlier test can leak `I18n.locale` and break a later assertion of
English validation messages. Do not hide this by choosing a passing seed or
weakening assertions. After an authorized baseline fix, rerun the failing seed
and the full suite before resuming the upgrade.

---

## Order of Operations

### 1. Fix load errors first

If the app cannot boot, nothing else matters. Identify and fix any:
- `LoadError` — a required file or constant is missing
- `NameError` — a constant was removed or renamed
- `NoMethodError` at load time — a method was removed from a class used in
  initializers

```bash
BUNDLE_GEMFILE=Gemfile.next bundle exec rails runner "puts 'boots OK'"
```

Common causes:
- Gem removed a class or module in a new version
- Rails removed a constant you were requiring (check version-specific notes)
- An initializer autoloads a constant that no longer exists

### 2. Fix errors before failures

Errors (exceptions raised during test execution) block entire test files and
mask further issues. Failures (assertion mismatches) are contained and
interpretable.

Run the suite, collect all errors, fix them as a batch before looking at
assertion failures.

### 3. Start with model tests

Model tests are the most isolated layer — no middleware, no routing, no views.
Getting model tests green first eliminates a category of cascading failures in
controllers, services, and integration tests.

```bash
# Minitest
BUNDLE_GEMFILE=Gemfile.next bundle exec rails test test/models

# RSpec
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec spec/models
```

### 4. Group failures by root cause

Resist fixing failures one by one. Look for patterns in error messages and
stack traces:

```bash
# Count unique error types
BUNDLE_GEMFILE=Gemfile.next bundle exec rails test 2>&1 \
  | grep "Error\|FAILED" | sort | uniq -c | sort -rn
```

Common groupings:
- All failures trace back to one changed API
- All failures involve a specific model or association
- All failures are in tests that mock a particular external service
- All failures involve a specific gem

Create a list of root causes and fix each group together.

---

## Debugging Techniques

### Compare against the current version

The dual-boot setup lets you run the same test in both Rails versions and diff
the results. This is the single most powerful debugging technique.

```bash
# Failing test in new version
BUNDLE_GEMFILE=Gemfile.next bundle exec rails test test/models/user_test.rb:42

# Same test passing in current version
bundle exec rails test test/models/user_test.rb:42
```

Insert a debugger in both contexts to compare variable state:

```ruby
binding.pry   # or: debugger
```

### Read the full error message

Rails error messages often include explicit migration hints. Examples:
- "Use `#with_connection` instead of `#connection`" — tells you exactly what to do
- "Calling `permit` on a Hash has been deprecated" — points to strong params code

Do not just read the exception class; read the full message and backtrace.

### Find the first line of your code in the backtrace

Skip Rails and gem frames. The first line that references your app code is
usually where the actual bug is.

```
# Example stack trace — look for your app's path
activerecord-8.0.0/lib/...        # skip
app/models/user.rb:34             # this is your code — start here
```

### Check version-specific notes

Before spending time debugging, cross-reference the error with
`references/version-specific-notes.md`. Many failures have known causes and
documented fixes.

### Revisit deprecation warnings

Deprecations that were warnings in the old version are hard errors in the new
one. If you did not fix every deprecation warning before starting the upgrade,
you will see them now as failures.

Run the old version with `ActiveSupport::Deprecation.behavior = :raise` to
catch any remaining ones:

```ruby
# test/test_helper.rb (temporarily, on the old version)
ActiveSupport::Deprecation.behavior = :raise
```

### Add logging to compare behaviour

When the cause is not obvious, add print statements to compare old vs new
runtime behaviour:

```ruby
Rails.logger.debug "DEBUG: value=#{value.inspect}"
```

Run both versions and compare the log output.

---

## Common Failure Patterns

### Strong parameters (various versions)

```
# Error
ActionController::UnpermittedParameters: found unpermitted parameter: ...
```

Check if `config.action_controller.action_on_unpermitted_parameters` was
changed. Also verify that `params.require().permit()` calls still cover
all submitted fields.

### Callback order changes

If tests that relied on `after_commit` or `after_save` callbacks now fail
with unexpected values, the default
`run_after_transaction_callbacks_in_order_defined` or
`run_commit_callbacks_on_first_saved_instances_in_transaction` may have
changed.

Check: `references/framework-defaults.md` for the specific config.

### Serialization issues (cache, cookies, messages)

```
# Error
ActiveSupport::MessageEncryptor::InvalidMessage
TypeError: no implicit conversion into String
```

These usually mean cached data or cookies were written by the old version
and are being read by the new version with a different serializer. Follow
the cache/cookie migration strategy in `references/framework-defaults.md`.

### Active Record query changes

In Rails 7.0+, `partial_inserts = false` means all columns are included in
INSERT statements, not just changed ones. This can change the behaviour of
tests that mock or assert on INSERT queries.

In Rails 7.1, `run_commit_callbacks_on_first_saved_instances_in_transaction = false`
changes which object receives commit callbacks when multiple instances of the
same record are modified in a transaction.

### Zeitwerk autoloading errors (6.x → 7.0)

```
# Error
NameError: uninitialized constant SomeClass
```

The `classic` autoloader was removed in Rails 7.0. Ensure:
1. All file names match their class/module names (Zeitwerk convention).
2. No `require` calls manually load autoloaded files.
3. No constants are autoloaded during initialisation outside `to_prepare`.

```bash
BUNDLE_GEMFILE=Gemfile.next bundle exec rails zeitwerk:check
```

### Mock / stub breakage

Mocks or stubs may rely on method signatures that changed in the new version.
For example:
- A method that previously accepted `nil` now raises
- A method was renamed or moved to a different module
- A method now returns a different type

Update test doubles to match the new signatures.

### Factory / fixture issues

If `ActiveRecord::Schema` changes (e.g. new `NOT NULL` columns with no
defaults) factories and fixtures may fail to create records. Add missing
attributes or set appropriate defaults.

---

## Working the Queue

Suggested daily workflow during a broken build phase:

1. Run the full suite. Record the error/failure count.
2. Pick the highest-frequency root cause.
3. Fix all instances of that root cause.
4. Run the suite again. Verify the count dropped.
5. Commit: `git commit -m "fix: <root cause description> after Rails X.Y bump"`
6. Repeat.

**Commit frequently.** Small, focused commits make it easy to bisect
regressions and give a clean rollback point if a fix introduces new failures.

---

## When You Are Stuck

1. Reduce to the smallest possible reproduction case.
2. Check the Rails issue tracker for the error message.
3. Check the gem's changelog/issues for Rails compatibility notes.
4. Swap to the old version with dual boot and add debugging to understand
   what the expected behaviour was.
5. Search the Rails CHANGELOG for the version:
   `https://github.com/rails/rails/blob/X-Y-stable/<component>/CHANGELOG.md`
