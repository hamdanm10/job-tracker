# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

This is a Rails 8.1 application (`JobTracker`) for tracking job applications, intended to run only on a
local server (single user, not deployed for public/multi-user access). Because of this, **no
authentication/authorization is needed** — do not add `devise`, `bcrypt`/`has_secure_password`, login
screens, or user-scoped access checks unless explicitly asked.

No domain models, controllers, or tests have been written yet. There is no `root` route configured. When
adding the first real feature, expect to also be setting conventions (test framework, first
models/controllers) that later work will follow.

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

## Architecture conventions

These conventions apply once real features are added (no services/repositories exist yet, so treat this
as the pattern to establish for the first one):

- **Service objects** (`app/services/`): put business logic/use cases here instead of in controllers or
  models. Every service inherits from a `ApplicationService` base class (`app/services/application_service.rb`)
  that exposes a `.call(...)` class method delegating to an instance `#call`, e.g.:
  ```ruby
  class ApplicationService
    def self.call(...) = new(...).call
  end

  class Users::CreateService < ApplicationService
    def initialize(params)
      @params = params
    end

    def call
      # ...
    end
  end
  ```
- **Repositories** (`app/repositories/`): put ActiveRecord query logic here instead of scattering
  `.where`/`.joins`/scopes across models, controllers, and services. One repository per model/aggregate,
  named `<Model>Repository` (e.g. `UserRepository`).
- **Routes** (`config/routes.rb`): always declare routes with `resources`/`resource` (with `only:`/
  `except:` and nested `member`/`collection` blocks as needed). Do not add bare `get`/`post`/`patch`/
  `delete` routes — `resources` keeps `routes.rb` short, RESTful, and easy to read.
- **Controllers** stay thin: only pass parameters into a service and render/redirect based on the
  result. Do not put business logic or ActiveRecord queries in controllers.
  - Business logic execution → **services** (`app/services/`).
  - Database access → **repositories** (`app/repositories/`), called from the service, not from the
    controller directly.
- **Reusable view components**: build them with the `view_component` gem instead of partials
  (`app/components/`, `<Name>Component < ApplicationComponent` with a matching `.html.erb` template).
  Generate new ones with `bin/rails generate view_component:component <Name> <args>` — it picks up
  `ApplicationComponent` (`app/components/application_component.rb`) as the base class automatically.
- **Pagination**: use the `pagy` gem for paginating data, not `kaminari`/`will_paginate` or manual
  `limit`/`offset`. `Pagy::Method` is included in `ApplicationController`, so call
  `pagy(:offset, scope, **options)` (or `:keyset`, etc.) in a controller action and render the result
  with the `@pagy` instance's helpers (e.g. `@pagy.series_nav`, `@pagy.info_tag`) in the view. Global
  options live in `config/initializers/pagy.rb`.
- **Avoid N+1 queries**: always eager-load associations used in loops/views (`includes`/`preload`/
  `eager_load` in the repository), and watch for N+1s introduced through service/view code.
- **Styling**: use Tailwind CSS (`tailwindcss-rails`) for all views/components, not the default
  `app/assets/stylesheets/application.css` handwritten CSS. `bin/dev` runs the Rails server together with
  the Tailwind watcher via `Procfile.dev`/foreman.

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

## Commit message conventions

When creating commits (and pushing to GitHub), follow the Angular-style convention
(see https://gist.github.com/nyancodeid/63f19941c81252bb0cca9c14497cf9f7):

```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<footer>
```

- The header (`<type>(<scope>): <subject>`) is mandatory; `<scope>` is optional. No line should
  exceed 100 characters.
- **Types**:
  - `build`: changes to the build system or external dependencies
  - `ci`: changes to CI configuration and scripts
  - `docs`: documentation-only changes
  - `feat`: a new feature
  - `fix`: a bug fix
  - `perf`: a code change that improves performance
  - `refactor`: a code change that neither fixes a bug nor adds a feature
  - `style`: changes that do not affect the meaning of the code (formatting, whitespace, etc.)
  - `test`: adding missing tests or correcting existing tests
- **Scope**: names the affected area of the app (e.g. a model, controller, or module). Leave it
  empty for changes that cut across scopes, such as broad `style`, `test`, or `refactor` commits.
- **Subject**: imperative, present tense ("add" not "added"/"adds"), no capitalized first letter,
  no trailing period.
- **Body**: same tense/style as the subject; explain the motivation for the change and contrast it
  with previous behavior.
- **Footer**: reference closed issues (e.g. `Closes #123`) and document breaking changes, prefixed
  with `BREAKING CHANGE:`.
- **Reverts**: start the subject with `revert:` followed by the header of the reverted commit, and
  include `This reverts commit <hash>.` in the body.
- **No Claude co-author**: do not add `Co-Authored-By: Claude` (or any Claude/Anthropic attribution)
  to commit messages.

## Notes

- `config.generators.system_tests = nil` — system test generators are disabled, so `rails g scaffold`/etc.
  won't produce system tests.
- Recurring Active Job schedules (Solid Queue) are defined in `config/recurring.yml`, currently only for
  `production`.
