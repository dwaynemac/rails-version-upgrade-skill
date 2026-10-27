# Framework Defaults (`config.load_defaults`) Migration

`config.load_defaults` is the mechanism Rails uses to let you opt into new
behavioural defaults one version at a time. Understanding how it works and
migrating it correctly is critical to a safe upgrade.

---

## How It Works

`config.load_defaults X.Y` loads the default configuration values for version
X.Y and *all previous versions*. For example, `config.load_defaults 8.0` also
loads all defaults introduced in 7.x, 6.x, etc.

When you upgrade Rails but leave `config.load_defaults` pointing at the old
version, your app continues to run with the old behavioural defaults even
though it is running on the new Rails code. This is intentional — it lets you
migrate defaults incrementally.

**The risk of not migrating defaults**: you can end up running on an unsupported
Rails version that you think is safe because it "works", but you are silently
skipping security and correctness improvements baked into new defaults.

---

## When to Migrate Defaults

There are two migration moments during an upgrade cycle:

1. **Before bumping the version** — align `config.load_defaults` to match
   the *current* Rails version. This catches any defaults you skipped in
   previous upgrades.

2. **After bumping the version and fixing the broken build** — work through the
   new `new_framework_defaults_X_Y.rb` initializer to adopt the new version's
   defaults.

---

## Step 1 — Check the Current State

```bash
grep "config.load_defaults" config/application.rb
```

If this value is lower than the current Rails version, you have skipped
defaults. Example: running Rails 7.2 but `config.load_defaults 7.0`.

You must migrate to `7.1` defaults before upgrading to `8.0`.

---

## Step 2 — Use `new_framework_defaults_X_Y.rb`

After running `bin/rails app:update`, Rails creates:

```
config/initializers/new_framework_defaults_X_Y.rb
```

This file contains all new defaults for the target version, **commented out**.
Uncommenting a line activates the new behaviour.

Use the target Rails gem's generated initializer and `load_defaults` source as
the inventory for the exact version being installed. The summaries below are
navigation aids, not a substitute for that inventory: an omitted setting can
otherwise first take effect when `config.load_defaults` is promoted, bypassing
its individual validation. Locate the selected gem with
`BUNDLE_GEMFILE=Gemfile.next bundle show railties` and inspect
`lib/rails/generators/rails/app/templates/config/initializers/` and
`lib/rails/application/configuration.rb`.

Workflow per default:
1. Read the inline comment explaining what the default changes.
2. Search the codebase for affected code.
3. Uncomment the line.
4. Run the test suite.
5. Fix any failures.
6. Commit.
7. Repeat for the next default.

When all defaults have been uncommented and the suite is green, update
`config.load_defaults` in `config/application.rb` and delete the initializer
file.

---

## Step 3 — Risk-Tier the Defaults

Not all defaults carry the same risk. Prioritise low-risk ones first to build
momentum, and tackle high-risk ones with more care.

### Low risk (safe to enable early)

These rarely break application code and are mainly performance or hygiene
improvements:

- `config.add_autoload_paths_to_load_path = false` (7.1)
- `config.precompile_filter_parameters = true` (7.1)
- `config.log_file_size` limit (7.1)
- `config.active_record.run_after_transaction_callbacks_in_order_defined` (7.1)
- `config.active_record.generate_secure_token_on = :initialize` (7.1)
- `Regexp.timeout = 1` (8.0)

### Medium risk (test carefully)

These change query or serialization behaviour and need verification:

- `config.active_record.query_log_tags_format = :sqlcommenter` (7.1)
- `config.active_support.cache_format_version` (7.0, 7.1)
- `config.active_record.marshalling_format_version = 7.1` (7.1)
- `config.active_record.postgresql_adapter_decode_dates = true` (7.2)
- `config.active_record.validate_migration_timestamps = true` (7.2)
- `config.action_dispatch.strict_freshness = true` (8.0)
- `config.active_record.sqlite3_adapter_strict_strings_by_default = true` (7.1)

### High risk (review manually before enabling)

These change security, encryption, session, or callback behaviour:

- `config.active_record.encryption.hash_digest_class` (7.1) — affects encrypted
  columns; see official guide before enabling
- `config.active_record.run_commit_callbacks_on_first_saved_instances_in_transaction = false` (7.1)
- `config.active_support.message_serializer = :json_allow_marshal` (7.1)
- `config.active_support.use_message_serializer_for_metadata = true` (7.1)
- `config.action_dispatch.cookies_serializer = :json` (7.0) — invalidates
  existing Marshal-serialized cookies; requires a rotation period

---

## Default Values by Version

### Rails 8.1 new defaults
- `config.action_controller.action_on_path_relative_redirect = :raise`
- `config.action_controller.escape_json_responses = false`
- `config.active_support.escape_js_separators_in_json = false`
- `config.action_view.render_tracker = :ruby`
- `config.action_view.remove_hidden_field_autocomplete = true`
- `config.active_record.raise_on_missing_required_finder_order_columns = true`
- `config.yjit = !Rails.env.local?`

### Rails 8.0 new defaults
- `Regexp.timeout = 1`
- `config.action_dispatch.strict_freshness = true`

### Rails 7.2 new defaults
- `config.active_record.postgresql_adapter_decode_dates = true`
- `config.active_record.validate_migration_timestamps = true`
- `config.active_storage.web_image_content_types` — adds `image/webp`
- `config.yjit = true`

### Rails 7.1 new defaults (key ones)
- `config.active_record.sqlite3_adapter_strict_strings_by_default = true`
- `config.active_record.encryption.hash_digest_class = OpenSSL::Digest::SHA256`
- `config.active_record.run_commit_callbacks_on_first_saved_instances_in_transaction = false`
- `config.active_record.query_log_tags_format = :sqlcommenter`
- `config.active_support.cache_format_version = 7.1`
- `config.active_support.message_serializer = :json_allow_marshal`
- `config.add_autoload_paths_to_load_path = false`
- `config.log_file_size = 100 * 1024 * 1024`

### Rails 7.0 new defaults (key ones)
- `config.action_dispatch.cookies_serializer = :json`
- `config.action_controller.action_on_open_redirect = :raise`
- `config.active_support.hash_digest_class = OpenSSL::Digest::SHA256`
- `config.active_support.key_generator_hash_digest_class = OpenSSL::Digest::SHA256`
- `config.active_record.partial_inserts = false`
- `config.active_record.verify_foreign_keys_for_fixtures = true`
- `config.active_storage.variant_processor = :vips`

For the full list, consult:
https://guides.rubyonrails.org/configuring.html#versioned-default-values

---

## Cache Format Migration (Rolling Deploys)

When upgrading to a version with a new cache format (7.0, 7.1), take care
with rolling deploys:

1. **First deploy**: do NOT change `config.active_support.cache_format_version`.
   New servers can read the old format.
2. **Second deploy**: enable the new cache format. Old servers will be replaced
   by this point and no longer read the cache.

Skipping step 1 in a rolling deploy causes cache misses on servers still
running the old code.

---

## Cookie Serializer Migration (JSON)

`config.action_dispatch.cookies_serializer = :json` (Rails 7.0 default) is a
high-risk change: existing cookies signed with the Marshal serializer will be
unreadable after the switch.

Migration path:
1. Add `:hybrid` as a transitional serializer (reads Marshal, writes JSON):

```ruby
# config/initializers/cookies_serializer.rb
Rails.application.config.action_dispatch.cookies_serializer = :hybrid
```

2. Deploy and let all active sessions naturally rotate to JSON.
3. After a suitable window (e.g. session lifetime + buffer), switch to `:json`.

---

## After Migrating All Defaults

Once all defaults in `new_framework_defaults_X_Y.rb` are uncommented and the
suite is green:

```ruby
# config/application.rb
config.load_defaults X.Y   # set to the new version
```

Delete `config/initializers/new_framework_defaults_X_Y.rb`.

Run the full suite one final time.
