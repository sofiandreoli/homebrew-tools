# Worktree Dev Manager (`wop`)

`wop` is a small CLI that spins up an **isolated development environment per Git branch** using **git worktrees**: a separate checkout, its own ports, generated `.env` files, databases, install/migrate hooks, and dev servers—without clobbering your main working tree.

Environments are scoped **per project** (the directory containing `.devmanager.yml`): two different repositories can have the same branch name without interfering, and `wop stop`/`restart`/`down` only ever touch the current project's environment. Worktrees are always created as siblings of the repository root, and `wop` commands work from any subdirectory of the project.

### Supported platforms

- **CLI (`wop`)**: macOS and Linux.
- **Desktop (Wopper)**: macOS (Apple Silicon and Intel).
- Windows is not supported yet (the process, port and terminal handling are Unix-specific).

### Versioning

The CLI and the desktop app are released separately (`wop` via Homebrew, `Wopper.app` via DMG) but embed the **same engine** and share one `registry.json`. They stamp a shared version string and record which binary last wrote the registry. Keep them on the **same minor version** — if they diverge, `wop` prints a one-line warning naming both versions. A registry written by a *newer* schema than your binary understands is never modified (upgrade the older side). The app's version is shown in its Help view, alongside a detected `wop` CLI version if one is installed.

---

## Why worktrees?

Normally, one repo means one checked-out branch at a time. Switching branches means stashing changes, waiting on installs, and losing your running dev server. **Git worktrees** break that constraint: each worktree is a full checkout of a different branch, sharing the same repo history, living at its own path on disk.

`wop` builds on this to give every branch a completely isolated environment — its own ports, `.env`, database, and dev server. Switch between features, hotfixes, or experiments the way you switch terminal tabs.

This is especially useful when running **AI coding agents in parallel** — without isolation, agents collide on ports, clobber each other's `.env`, and corrupt shared databases. With `wop`, each gets its own lane.

---

## What `wop` does for you

From the **repository root** (where `.devmanager.yml` lives), a typical:

```bash
wop up feature/login
```

will:

1. **Create a worktree** for `feature/login` next to your main repo (see [Worktree layout](#worktree-layout)).
2. **Allocate a free port** per service (non-overlapping ranges are enforced in the config).
3. **Write `.env` files** in the worktree: a shared one at the worktree root (optional) and per-service ones under each service `dir` (optional), substituting [template variables](#template-variables-placeholders).
4. **Create databases** for every logical database entry in the config (as part of `wop up`).
5. Run **`after_create` hooks** per service (e.g. `bundle install`, `rails db:migrate`, `npm install`).
6. **Start each service** (`cmd`) in its directory, with env loaded from the worktree `.env` and the service `.env` (see [Environment loading at runtime](#environment-loading-at-runtime)).
7. **Register PIDs and ports** so `wop list` and `wop down` know what to stop.

`wop down <branch>` stops registered processes, removes the worktree, drops the branch databases, and clears the registry entry.

---

## Committing and pushing from a worktree

A worktree is a full checkout — `git` works exactly as you'd expect. Just `cd` into the worktree directory and use your normal flow:

```bash
git add .
git commit -m "your message"
git push origin my-feature-branch
```

The branch is already set, so no extra flags needed. You're committing directly to that branch without touching your main tree.

---

## Requirements

- **Git** (worktrees).
- **Go 1.22+** only if you build from source.
- **Database CLI tools** for the adapters you use (e.g. `createdb` / `dropdb` / `psql` for PostgreSQL).

Run all `wop` commands from the **project root** that contains `.devmanager.yml`.

---

## Install (Homebrew)

```bash
brew install sofiandreoli/tools/wop
wop --version
```

---

## Commands

| Command                               | Description                                                                                                    |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `wop up <branch>`                     | Create worktree, generate env files, create DBs, run `after_create` hooks, start services, register processes. |
| `wop down <branch>`                   | Stop services (from registry), remove worktree, drop DBs, remove registry entry.                               |
| `wop stop <branch>`                   | Kill (`-9`) all services for a branch. Keeps the worktree, databases, and registry entry (PID cleared).        |
| `wop stop <branch> <service>`         | Kill (`-9`) a single service. Same as above but scoped to one service.                                         |
| `wop restart <branch>`                | Restart all stopped services for a branch (re-uses stored `cmd` and port).                                     |
| `wop restart <branch> <service>`      | Restart a single stopped service.                                                                              |
| `wop list`                            | List active environments (branch, service, port, PID, status).                                                 |
| `wop cleanup`                         | Remove stale (stopped) entries from the registry; drops worktree and DBs if all services for a branch stopped. |
| `wop config show`                     | Print the parsed `.devmanager.yml` for the current directory.                                                  |
| `wop ports scan`                      | Print the first free port in each service’s `port_range`.                                                      |
| `wop --version`                       | Print the embedded version string.                                                                             |
| `wop --help`                          | Short usage (same text as below).                                                                              |

---

## Worktree layout

Paths are resolved from your **current working directory**. The parent directory of the repo root, then `{app.name}--{branch-with-slashes-as-hyphens}`.

Example: if the app is `my-first-app`, branch `feature/login`, and you run `wop` from `/projects/my-first-app`, the worktree path is `/projects/my-first-app--feature-login`.

---

## `.devmanager.yml`

Place this file at the **root of the project** you run `wop` from (alongside your main `.git`).

### Example

```yaml
app:
  name: my-first-app

services:
  env_source: .env
  env:
    PORT: "{backend_port}"
    DATABASE_URL: "{database_url_primary}"
    DB_NAME_TEST: "{db_name_test}"

  backend:
    dir: my-first-app-api
    cmd: bundle exec rails s
    port_range: [3000, 3099]

  frontend:
    dir: my-first-app-web
    cmd: npm run dev
    port_range: [5173, 5200]

databases:
  primary:
    adapter: postgresql
    name_pattern: "{app}_{branch_slug}"
    migrate_on_up: false
  test:
    adapter: postgresql
    name_pattern: "{app}_{branch_slug}_test"
    migrate_on_up: false

hooks:
  backend:
    after_create:
      - bundle install
      - rails db:migrate
      - rails db:seed
      - rails db:test:prepare
  frontend:
    after_create:
      - npm install
```

## `app`

| Field  | Meaning                                                                                                |
| ------ | ------------------------------------------------------------------------------------------------------ |
| `name` | Used for the worktree directory name and for default tokens like `{app}` / `{app_name}`. **Required.** |

### `services`

You need at least one **named service** (`backend`, `frontend`, …). Each has `cmd`, `port_range`, and optional `dir`.

**Environment files — two places (you can use one, the other, or both):**

1. **App-wide / shared** — put `env_source` and/or `env` directly under `services` (not under a service name). `wop` writes **`<worktree>/.env`**.
2. **Per service** — put `env_source` and/or `env` under that service (e.g. under `backend:`). `wop` writes **`<worktree>/<dir>/.env`** for that service (or the worktree root if `dir` is omitted or `.`).

**How a file is built:** `wop` starts from **`env_source`** (if set): it reads that file from disk (path is relative to **where you run `wop`**, usually the main repo root), expands `{placeholders}` in the values, and keeps every line as the baseline env. Then it applies the **`env`** map from YAML: for each key, it **sets or replaces** that variable. Anything in `env` that was already in the file is **overwritten**; keys only in `env` are **added**. So the YAML `env` block is always a patch on top of the template file (or an empty baseline if you only use `env` and skip `env_source`).

#### Service fields

| Field        | Required | Meaning                                                                                                                |
| ------------ | -------- | ---------------------------------------------------------------------------------------------------------------------- |
| `dir`        | No       | Subdirectory inside the worktree where the command runs (e.g. `eagerpm-api`). Use `.` or omit for the worktree root.   |
| `cmd`        | **Yes**  | Shell command to start the service. It runs through `sh -c`, so `$VARS`, `&&`, pipes and quoted arguments all work. Logs go to `<dir>/<service-name>.log`. |
| `port_range` | **Yes**  | `[low, high]` inclusive. `wop` picks the first free port in that range. Ranges **must not overlap** between services.  |
| `env_source` | No       | Template env file (relative to cwd when you run `wop`). See [Template variables](#template-variables-placeholders).    |
| `env`        | No       | Key/value pairs patched onto the result of `env_source` (see above).                                                   |

### `databases`

A map of **logical** names (e.g. `primary`, `test`):

| Field           | Meaning                                                                                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `adapter`       | One of the supported adapters (see below).                                                                                                                                                |
| `name_pattern`  | Pattern for the physical DB name. Placeholders: `{app}`, `{app_name}`, `{branch}`, `{branch_slug}` (`{branch}` and `{branch_slug}` are synonyms here) — see [Database `name_pattern` vs env `{branch_slug}`](#database-name_pattern-vs-env-branch_slug). An unknown `{token}` is a hard error. |
| `copy_from`     | Optional. Name of an **existing** database to clone instead of creating an empty one — see [Copying a database](#copying-a-database). Used literally (no placeholder expansion).          |
| `migrate_on_up` | Parsed and shown in `wop config show`; **not automatically run by `wop` yet** — run migrations in `after_create` hooks if you need them.                                                  |

Supported **adapter** names today:

| Adapter value | Notes                                 |
| ------------- | ------------------------------------- |
| `postgresql`  | Uses `createdb`, `dropdb`, `psql`.    |
| `mongodb`     | Mongo creation/drop.                  |
| `sqlite`      | File-based SQLite under the worktree. |
| `redis`       | Namespaced keys via `redis-cli`.      |

The short aliases `postgres`, `mongo` and `sqlite3` are also accepted for backward compatibility, but the canonical names above are recommended.

#### Database connection (`connection`)

By default `wop` connects to a database server on `localhost` (and respects the adapter's standard environment variables such as `PGHOST`/`PGPORT`/`PGUSER`). To point at a non-default host, port or user, add an optional `connection` block to a logical database:

```yaml
databases:
  primary:
    adapter: postgresql
    name_pattern: "{app}_{branch_slug}"
    connection:
      host: localhost
      port: 5433
      user: postgres
      password_env: MY_PG_PASSWORD   # NAME of an env var, never the password itself
```

- Every field is optional, and the whole block is optional. **Omitting it behaves exactly as before.**
- `password_env` names an environment variable read at runtime — `.devmanager.yml` is committed, so never put a plaintext password here. If the named variable is unset, `wop` errors and names it.
- Resolution precedence is `connection` block → the adapter's standard env vars → built-in defaults. The **same** resolved connection builds both the CLI flags used to create the database and the URL written into `.env`, so creating and connecting can never disagree.
- `sqlite` has no server; a `connection` block on a sqlite database is a validation error.
- **`redis` does NOT isolate per branch automatically.** All branches share the same server, and `{database_url_<logical>}` is that shared address — the URL carries no namespace. Isolation is a **convention you must follow**: prefix every key you write with `{db_name_<logical>}` (which resolves to a per-branch value like `myapp_feature_login_9f2a`). `wop down`/`cleanup` then delete exactly `<that-prefix>:*` and a `__wop:<prefix>` sentinel. If you don't prefix your keys, branches will share data and `wop down` won't clean them up.

#### Database `name_pattern` vs env `{branch_slug}`

The same token name, `{branch_slug}`, is sanitized **differently** depending on where it is used, because database names and env values have different rules:

- **In env values** (`services.env`, per-service `env`, `env_source` files), `{branch_slug}` maps `/` → `-` and preserves case: `feature/Login` → `feature-Login`.
- **In a database `name_pattern`**, `{branch}` and `{branch_slug}` are synonyms and produce a database-safe slug: lowercased, with `/` and `-` both mapped to `_`. Because that mapping is not reversible (`feature/login` and `feature-login` would both become `feature_login`), a short hash of the original branch name is appended **whenever** slugifying changed the string. Simple names such as `main`, `develop` and `staging` are left untouched.

So `feature/login` yields the env slug `feature-login` but a database name like `myapp_feature_login_9f2a3b1c`. This keeps two different branches from silently sharing one database; it makes collisions improbable, not impossible.

#### Copying a database

By default `wop` creates an **empty** database for each worktree (you then seed it via `after_create` hooks). Sometimes you'd rather start from a copy of an existing database — e.g. an app `CarOne` with a `car_one_development` database full of real data. Set `copy_from` to that database's name and `wop` clones it into the name `name_pattern` would have created:

```yaml
databases:
  primary:
    adapter: postgresql
    name_pattern: "{app}_{branch_slug}"
    copy_from: car_one_development
```

`copy_from` is used **literally** — no `{app}` / `{branch_slug}` expansion — so write the exact source database name. How the clone happens per adapter:

| Adapter      | How `copy_from` is cloned                                                                                  |
| ------------ | ---------------------------------------------------------------------------------------------------------- |
| `postgresql` | `createdb` the target, then stream `pg_dump <source> \| psql <target>` (works while the source is in use). |
| `mongodb`    | `mongodump --archive` from the source piped into `mongorestore` with the namespace remapped to the target. |
| `sqlite`     | Copies the source database file to the target path.                                                        |
| `redis`      | Copies every `source:*` key to `target:*` with `redis-cli COPY`.                                           |

If the target already exists `wop` skips it; if the source is missing it errors. Custom adapters support copying too via a `copy` command (see below).

#### Custom adapters

If your database isn't supported, define a custom adapter under `custom_adapters`. Provide shell commands for `create`, `drop`, and `url` — use `{db_name}` as the placeholder for the resolved database name. Then reference it by name in `databases.<logical>.adapter`. Add an optional `copy` command (with `{source_db_name}` and `{db_name}`) to support `copy_from`.

```yaml
custom_adapters:
  mysql:
    create: mysql -e "CREATE DATABASE {db_name}"
    drop: mysql -e "DROP DATABASE IF EXISTS {db_name}"
    exists: mysql -e "USE {db_name}" 2>/dev/null  # optional
    copy: mysql -e "CREATE DATABASE {db_name}" && mysqldump {source_db_name} | mysql {db_name}  # optional, enables copy_from
    url: mysql://root@localhost/{db_name}

databases:
  primary:
    adapter: mysql
    name_pattern: "{app}_{branch_slug}"
```

### `hooks`

Hooks are grouped **by service name** (the same keys as under `services`). Only hooks for **defined services** are accepted.

| List           | When it runs (current behavior)                                                                                                          |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `after_create` | After DB creation, before services start. Commands run in the service `dir` with [hook environment](#hook-environment). **Implemented.** |
| `after_up`     | After all services have started successfully. **Implemented.**                                                                          |
| `before_down`  | Before `wop down` tears the environment down. **Implemented.**                                                                          |

---

## Template variables (placeholders)

In `services.env`, per-service `env`, and in values inside files referenced by `env_source`, `wop` replaces `{token}` with runtime values.

### Always useful

| Token           | Description                                                     |
| --------------- | --------------------------------------------------------------- |
| `app_name`      | Same as `app.name`.                                             |
| `app`           | Same as `app.name`.                                             |
| `branch`        | Git branch name (e.g. `feature/login`).                         |
| `branch_slug`   | Branch with `/` → `-` (e.g. `feature/login` → `feature-login`). |
| `worktree_path` | Absolute path to the worktree root.                             |

### Ports for every service

For each service key `<name>`, `wop` exposes `{<name>_port}` to **all** generated files for that run (e.g. `backend` → `{backend_port}`). Use them to wire API URLs, CORS, etc.

### Databases

For each logical database key `<logical>` in `databases`:

| Token                    | Description                                               |
| ------------------------ | --------------------------------------------------------- |
| `database_url_<logical>` | Connection URL for that DB (e.g. `database_url_primary`). |
| `db_name_<logical>`      | Resolved database name (e.g. `db_name_test`).             |

Logical keys are sorted alphabetically when operations run; naming is up to you (`primary`, `test`, `cache`, …).

---

## Registry

`wop` stores active runs under **`~/.config/devmanager`** so `wop list` and `wop down` can find PIDs and ports.
