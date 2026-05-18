# Worktree Dev Manager (`wop`)

`wop` is a small CLI that spins up an **isolated development environment per Git branch** using **git worktrees**: a separate checkout, its own ports, generated `.env` files, databases, install/migrate hooks, and dev servers—without clobbering your main working tree.

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
brew install sofiandreoli/tap/wop
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
| `cmd`        | **Yes**  | Shell command to start the service (split on spaces for the executable + args). Logs go to `<dir>/<service-name>.log`. |
| `port_range` | **Yes**  | `[low, high]` inclusive. `wop` picks the first free port in that range. Ranges **must not overlap** between services.  |
| `env_source` | No       | Template env file (relative to cwd when you run `wop`). See [Template variables](#template-variables-placeholders).    |
| `env`        | No       | Key/value pairs patched onto the result of `env_source` (see above).                                                   |

### `databases`

A map of **logical** names (e.g. `primary`, `test`):

| Field           | Meaning                                                                                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `adapter`       | One of the supported adapters (see below).                                                                                                                                                |
| `name_pattern`  | Pattern for the physical DB name. Placeholders: `{app}`, `{app_name}`, `{branch_slug}` — see [Database `name_pattern` vs env `{branch_slug}`](#database-name_pattern-vs-env-branch_slug). |
| `migrate_on_up` | Parsed and shown in `wop config show`; **not automatically run by `wop` yet** — run migrations in `after_create` hooks if you need them.                                                  |

Supported **adapter** names today:

| Adapter value | Notes                                 |
| ------------- | ------------------------------------- |
| `postgresql`  | Uses `createdb`, `dropdb`, `psql`.    |
| `mongodb`     | Mongo creation/drop.                  |
| `sqlite`      | File-based SQLite under the worktree. |

### `hooks`

Hooks are grouped **by service name** (the same keys as under `services`). Only hooks for **defined services** are accepted.

| List           | When it runs (current behavior)                                                                                                          |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `after_create` | After DB creation, before services start. Commands run in the service `dir` with [hook environment](#hook-environment). **Implemented.** |

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
