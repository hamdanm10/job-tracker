# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a freshly generated Rails 8.1 application (`JobTracker`) — no domain models, controllers, or tests
have been written yet. There is no `root` route configured. When adding the first real feature, expect to
also be setting conventions (test framework, first models/controllers) that later work will follow.

## Stack

- **Rails 8.1**, Ruby version pinned in `.ruby-version`.
- **SQLite** for all environments (`config/database.yml`). Production splits into four separate SQLite
  databases: `primary`, `cache`, `queue`, `cable` (each with its own `db/*_schema.rb` / migration path).
- **Propshaft** for assets, **importmap-rails** for JS (no bundler/Node build step).
- **Hotwire**: Turbo + Stimulus. Stimulus controllers live in `app/javascript/controllers/` and are
  auto-registered via `app/javascript/controllers/index.js`.
- **Solid trio** for infra, backed by the dedicated SQLite DBs above: `solid_queue` (Active Job backend),
  `solid_cache` (Rails.cache), `solid_cable` (Action Cable).
- No test framework is installed yet — `rails/test_unit/railtie` is explicitly commented out in
  `config/application.rb`, and there is no `test/` or `spec/` directory. Decide/confirm the test setup
  before assuming `bin/rails test` or `rspec` works.
- Deployment via **Kamal** (`config/deploy.yml`, `.kamal/`) to a Docker container; **Thruster** in front of
  Puma in production.

## Commands

Setup:
```
bin/setup              # bundle install, db:prepare, clear logs/tmp, then starts the dev server
bin/setup --skip-server
bin/setup --reset      # also runs db:reset
```

Run the dev server:
```
bin/dev                # runs bin/rails server
```

Console / DB:
```
bin/rails console
bin/rails db:prepare
bin/rails db:migrate
```

Full CI suite (mirrors `.github/workflows/ci.yml`, driven by `config/ci.rb`):
```
bin/ci
```

Individual checks (each is also its own CI job/step):
```
bin/rubocop                 # Ruby style (rubocop-rails-omakase); add -A to autocorrect
bin/brakeman --no-pager     # static security analysis
bin/bundler-audit           # gem vulnerability audit
bin/importmap audit         # JS dependency vulnerability audit
```

There is currently no test-running command — see "Project state" above.

## Notes

- `config.generators.system_tests = nil` — system test generators are disabled, so `rails g scaffold`/etc.
  won't produce system tests.
- Recurring Active Job schedules (Solid Queue) are defined in `config/recurring.yml`, currently only for
  `production`.
