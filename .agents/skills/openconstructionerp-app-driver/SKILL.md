---
name: openconstructionerp-app-driver
description: Operate and verify the OpenConstructionERP application through Docker and browser automation. Use for starting or discovering the app, login, navigation, UI workflows, smoke tests, and end-to-end troubleshooting.
compatibility: Requires Docker, curl, and a browser automation interface when performing UI workflows. Credentials must be supplied by the user or an approved environment.
metadata:
  author: opentechexpert
  version: "1.2"
---

# OpenConstructionERP app driver

Use this skill when an agent needs to operate the running OpenConstructionERP application rather than only edit source code. The browser automation interface may be a browser tool, Playwright, WebDriver, a browser canvas, or another harness-specific adapter.

## Operating principles

- Prefer the already-running application over starting a second copy.
- Host the stack in the repository's main checkout so it outlives any single worktree or session.
- Never print, commit, or expose passwords, tokens, cookies, or session headers.
- Do not invent credentials. Ask the user for credentials or use credentials explicitly provided through the environment.
- Treat destructive actions as requiring confirmation unless the user explicitly requested them.
- Verify every important action through the UI or an application/API health check; do not claim success from a click alone.
- A silent UI is not a failed action. Confirm long-running work in the container log and the API before retrying it.
- Keep browser state and application state separate: a successful page load does not prove that login or a business workflow succeeded.

## 1. Discover the application

From the repository root, inspect running containers before starting anything:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
```

Identify the application container by its image/name and use its published host port. Common deployments publish the production container on port `8080`, while local Vite development commonly uses `5173`. Do not assume either port when `docker ps` provides a different mapping.

Check readiness using the published URL:

```bash
curl --fail --silent --show-error --max-time 10 http://localhost:<published-port>/
```

If the application container is absent, inspect the available compose files and use the project’s documented quickstart or production command. Before starting containers, check for an existing stack and avoid disrupting unrelated containers:

```bash
docker compose ps
```

Use `docker compose up -d` only when the user asked to start the application or no suitable application container is running. After startup, wait for the container health status and re-check the published URL.

### Confirm application health, not just HTTP reachability

A `200` from `/` only proves the frontend shell was served. Use the application health endpoint to confirm the backend and database are actually ready:

```bash
curl --fail --silent --max-time 10 http://localhost:<published-port>/api/health
```

A healthy response reports `status: healthy` plus `version`, `modules_loaded`, and `database: ok`. `schema_matches_models` can be `true` or `null` on a healthy response; treat only `schema_matches_models: false` or a non-`ok` database as a blocker and report it instead of proceeding with UI work.

### Run the stack from the main checkout, not a worktree

Host the stack in the repository's **main checkout**, not in a throwaway worktree. A worktree is deleted when its session ends, which strands the stack: the containers keep running under a compose project whose directory no longer exists, and the data sits in volumes named after a branch nobody recognises.

The compose project name defaults to the directory name, so running from the main checkout gives a stable, predictable project (`openconstructionerp`) and volumes (`openconstructionerp_pg_data`, `openconstructionerp_app_data`).

```bash
git worktree list          # first entry is the main checkout
cd <main-checkout>
make quickstart-arm64      # or the platform-appropriate target
```

`docker ps` may still reveal a healthy stack started from a *different* worktree. `docker compose` run from the current directory will not control it — target it by project name instead:

```bash
docker ps --format '{{.Names}}' # project is the name prefix
docker inspect <container> \
  --format '{{index .Config.Labels "com.docker.compose.project"}}'
docker compose -f <same-compose-file> -p <project> down   # no -v: keeps the named volumes
```

The `com.docker.compose.project.working_dir` and `com.docker.compose.project.config_files` labels record where the stack was launched from and which files built it. Note the dots: a `project_working_dir` spelling matches nothing and returns empty, which reads as "no directory" rather than "wrong key".

```bash
docker inspect <container> --format \
  '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
```

Prefer the project name to address the stack, and reuse the original Compose file list when running Compose commands against it.

### Relocating a running stack without losing data

Two things bind the data to the old project, and both must be carried over:

1. **`.env`** — `make quickstart-secrets` only checks for `POSTGRES_PASSWORD` and `JWT_SECRET`, then prints commands and exits when either is missing. The password is baked into the Postgres volume at initialisation, so a freshly generated `.env` cannot read an existing volume. Copying the old `.env` first is both safe and required.
2. **Named volumes** — they are prefixed with the project name and do not follow a move.

```bash
cp -p <old-dir>/.env <main-checkout>/.env
docker compose -f <same-compose-file> -p <old-project> down          # no -v
for v in pg_data app_data; do
  docker volume create "<new-project>_$v"
  docker run --rm -v "<old-project>_$v":/from -v "<new-project>_$v":/to \
    alpine sh -c 'cd /from && cp -a . /to/'
done
cd <main-checkout> && make quickstart-arm64
```

Compose warns that the volumes were "not created by Docker Compose"; that is expected for pre-seeded volumes and is not an error. Verify the migration by confirming `/api/health` is healthy **and** that a known user still logs in with the same id — a healthy stack on an empty database looks identical to a successful move. Keep the old volumes until that check passes, then remove them.

### Apple Silicon (arm64) hosts

The published image is amd64 and runs under emulation on arm64. The no-clone quickstart Docker commands in `docs/getting-started.md` do not work on Apple Silicon. Clone the repository and use the arm64 path instead:

```bash
make quickstart-arm64   # uses docker-compose.arm64.yml
```

This target requires a `.env` containing `POSTGRES_PASSWORD` and `JWT_SECRET`; it fails with instructions if one is absent, rather than shipping defaults.

### Do not expect to run the backend test suite locally

The backend requires Python `>=3.12`. A macOS host commonly only has the system `/usr/bin/python3` (3.9.x) with no `pytest`, in which case `pip install -e ./backend` fails on the version gate and editable-install backend. Do not burn turns creating throwaway virtualenvs. Verify behaviour against the running container instead, and say plainly that the suite was not executed rather than implying it passed.

## 2. Open and inspect the app

Open the discovered URL in the available browser automation interface. For the standard container deployment:

```text
http://localhost:8080/
```

If the app redirects to `/login`, that is expected. Use the harness’s structured browser operations where available:

1. Read or inspect the current page to identify its state and interactive elements.
2. Click or type using stable element references, labels, roles, or selectors.
3. Re-inspect after navigation, form submission, or asynchronous UI updates.
4. Capture a screenshot only when visual layout or an error banner cannot be confirmed from page text.

Do not use arbitrary JavaScript evaluation as the default interaction mechanism. Use it only for diagnostics when structured browser actions cannot inspect the needed state.

### Element references go stale

Element references (`e1`, `e8`, …) are invalidated by any navigation, route change, or form submission, and also by an in-place re-render such as a wizard advancing to its next step without changing the URL. Re-read the page for fresh references before every `type`/`click` that follows any state change, otherwise the action fails with a stale-reference error.

Re-reading also matters because a button's *meaning* can change while its reference stays valid. On the onboarding data step, selecting a different country rewrites the install button's label in place, so the same reference installs a different pack. Confirm the label reads what you expect before clicking a destructive or slow action.

### Routes behave differently depending on session state

- `/register` and `/users` redirect to `/dashboard` while a session is active. To reach the public registration form, log out or wait for the session to expire, then use the **Create account** link on `/login`.
- An expired session redirects to `/login?next=<original-path>` and resumes there after login.
- A brand-new real admin lands on the 6-step setup wizard at `/onboarding`, not `/dashboard`. This is correct behaviour, not a failure — the demo accounts skip it because they are pre-onboarded. See "Driving the first-run setup wizard" for how to handle it.
- Completing onboarding exits to `/projects`. The wizard reappears on later logins until it is completed or skipped, which is not evidence of data loss.

## 3. Authenticate safely

Determine whether the page is a login screen before entering anything. If credentials are not already available from an approved environment variable or explicit user instruction, ask the user for them.

- Enter credentials only into the application’s visible login fields.
- Do not echo the values in tool arguments, logs, summaries, screenshots, or commits when the tool surface would expose them.
- Submit the form, then inspect the page to verify that the authenticated application shell, dashboard, or requested destination is present.
- If login fails, report the visible error and stop; do not repeatedly guess credentials.
- If MFA, SSO, CAPTCHA, or an approval prompt appears, pause and ask the user to complete it.

### Demo accounts

Seeded demo accounts use the `@openconstructionerp.com` domain. Fresh installs store a randomly generated password per installation unless `DEMO_*_PASSWORD` is set, so the stored hash never matches the documented `DemoPass1234!`.

On a non-production install with `SEED_DEMO` enabled, the login form routes demo emails through a password-free shortcut, so **any** password submitted for a demo address succeeds. That makes the documented credentials work without a hardcoded password, but it also means a successful demo login proves nothing about the password you typed. Do not use a demo account to test authentication itself. Production installs set `SEED_DEMO=false`, which disables the shortcut and restores normal password verification.

Prefer the login page's one-click demo sign-in button or `/auth/demo-login/`, which are the fastest ways to smoke-test without typing a password.

Demo accounts are not a substitute for a real account. When the user asks for a "real" user, do not hand them a demo login.

### Handling a generated password

If the user asks you to generate a password, keep it out of tool arguments and logs:

```bash
umask 077 && python3 -c "import secrets,string; \
  alphabet=string.ascii_letters+string.digits+'+/'; \
  chars=[secrets.choice(string.ascii_letters), secrets.choice(string.digits)] + [secrets.choice(alphabet) for _ in range(18)]; \
  secrets.SystemRandom().shuffle(chars); print(''.join(chars))" > "${TMPDIR:-/tmp}/pw.txt"
```

Pass it to the API by reading the file *inside* the request script, never by interpolating it into a command line. Reveal it to the user only if they explicitly asked for a one-time display, tell them to change it immediately, and delete the file when done:

```bash
shred -u "${TMPDIR:-/tmp}/pw.txt" 2>/dev/null || rm -f "${TMPDIR:-/tmp}/pw.txt"
```

## 4. API access and user administration

### Endpoint layout

Auth endpoints are nested **under the users module**, not at a top-level `/auth`. The trailing-slash forms are canonical and visible in OpenAPI; bare forms are also registered for compatibility but hidden from the spec.

```text
POST /api/v1/users/auth/login/
POST /api/v1/users/auth/register/
POST /api/v1/users/auth/refresh/
POST /api/v1/users/auth/demo-login/
GET  /api/v1/users/me/
GET  /api/v1/users/            # users.list permission (manager+) required
```

Guessing `/api/v1/auth/login` returns `404`. When a path is unknown, read the spec rather than guessing — but note it is several megabytes, so filter it and never dump it into the transcript:

```bash
curl -s http://localhost:<published-port>/api/openapi.json \
  | python3 -c "import json,sys; print('\n'.join(p for p in json.load(sys.stdin)['paths'] if 'auth' in p))"
```

### Creating the first real admin

The installation bootstraps its first administrator through normal registration:

- `UserRepository.has_admin()` deliberately excludes any email matching `%@openconstructionerp.com`, so a demo-seeded install still reports "no admin".
- `UserService.register()` therefore promotes the **first registrant with a non-demo email** to `role="admin"`, `is_active=True`. This bootstrap path is permitted even when `registration_mode` is `closed`, so an operator can always get in.
- Every later registrant defaults to `OE_DEFAULT_REGISTRATION_ROLE` (`viewer` unless overridden to `editor` or `manager`); `admin` is never grantable through self-registration.

So the supported way to create a real admin is the ordinary registration endpoint or the `/register` form — not direct database edits. Confirm no real admin exists first, since the promotion only applies to the first one.

Password policy (`backend/app/modules/users/schemas.py`): at least one letter and one digit, not in the common-password blacklist, minimum 8 characters for public registration and 12 for the admin-created-user endpoint.

### Verifying a created user

A `201` response alone is not proof the account is usable. Confirm all of:

1. `POST /api/v1/users/auth/login/` returns a token.
2. `GET /api/v1/users/me/` reports the expected `role` and `is_active: true`.
3. For an admin, an admin-only endpoint such as `GET /api/v1/users/auth/demo-login/settings/` returns `200` rather than `403`.
4. The credentials work in the browser login form and reach an authenticated route.

## 5. Execute app workflows

Before changing data, identify whether the requested workflow is read-only or mutating. For a mutating workflow:

1. Navigate to the requested module through visible UI controls.
2. Confirm the destination page and relevant record/context.
3. Fill only the requested fields.
4. Review the form state before submitting.
5. Submit once and wait for the application response.
6. Verify the success notification, updated record, or resulting page.

For read-only investigation, prefer navigation and filtering over direct database manipulation. Use application APIs only when the user asks for API-level testing or the UI cannot expose the needed diagnostic.

### Long-running actions report nothing in the UI

Some actions start a streaming server-side job and give no spinner, no toast, and no progress bar. The trigger button simply disappears from the DOM, which is indistinguishable from a click that did nothing. Installing a partner pack (`POST /api/v1/partner-pack/full-install-stream`) behaves this way: it downloads a cost-data parquet of tens of megabytes and imports it for well over a minute while the page looks idle.

Do not retry the click and do not report failure. Confirm against the container log, then against the API:

```bash
docker logs --since 10m <application-container> 2>&1 \
  | grep -iE 'pack|import|positions|error'
```

The import logs its own completion, for example `CWICR USA_USD: 55719 imported, 0 skipped in 78.4s`.

Counters already rendered on the page are **stale client state** and are not re-fetched when the job finishes. A panel reading `0 loaded` after a successful import is a display artefact, not evidence. Verify with the API instead:

```bash
GET /api/v1/partner-pack/installed   # active_slug plus the installed list
GET /api/v1/costs/base-catalog/      # per-base loaded counts
```

### Driving the first-run setup wizard

A newly created real admin lands on `/onboarding`, a six-step wizard: Welcome (language), Start, Profile, Modules, Data, Finish. Every step offers a **Skip setup** control that jumps straight to the dashboard.

These steps encode business decisions, so present the options and let the user choose rather than picking defaults:

- **Start** offers Quick Start, a ready-made country pack, or choosing a profile by hand.
- **Profile** sets a module baseline by team size, from Solo (5 modules) to Large Enterprise (67).
- **Modules** only controls what appears in the menu. Nothing is deleted and it is changeable in Settings, so it is a safe choice to revisit.
- **Data** is the only step that loads data, and it is the slow one. A country pack installs language, cost databases and sample projects in one action. It **downloads from GitHub at install time**, so it needs network egress from the container and is not available on an air-gapped host.

The wizard exits to `/projects`, not `/dashboard`. Treat arrival there as the completion signal and confirm the module navigation rendered.

## 6. Smoke-test checklist

For a basic availability check, verify:

- The application container is running and healthy.
- The published root URL responds successfully.
- `/api/health` reports `status: healthy`, `database: ok`, and does not report `schema_matches_models: false`.
- The login page renders without a server error.
- Static assets load and the page has no obvious fatal error.

For an authenticated smoke test, additionally verify:

- Login completes.
- The main navigation or dashboard renders.
- One representative read-only page opens.
- Logout returns to the login page, unless the user asked to preserve the session.

Report the exact URL, container/port discovered, workflow steps performed, and any blocker. Do not report a workflow as complete without a post-action verification.

## 7. Troubleshooting

Use these checks in order:

```bash
docker ps
docker compose ps
docker logs --tail 100 <application-container>
curl --fail --silent --show-error http://localhost:<published-port>/
```

Common distinctions:

- `localhost:5173` is usually a development server and is not the correct URL for a production Docker container.
- A healthy container with an unreachable host URL usually indicates a missing or different port mapping.
- A reachable login page with failed login indicates an authentication or seed-data issue, not a Docker networking issue.
- A page that loads but lacks expected content may indicate frontend asset, API proxy, or backend readiness problems; inspect browser errors and container logs before changing code.
- A `404` from an API call is usually a wrong path, not a missing feature — check the nesting and OpenAPI-listed path before concluding the endpoint does not exist.
- A stale-element-reference error is a browser-harness issue, not an application bug; re-read the page and retry once.
- A button that vanishes after a click without a spinner or toast usually means a streaming server-side job started. Check the container log before retrying; a second click can start a duplicate import.
- An on-screen counter that still reads zero after an import finished is stale client state. Re-fetch from the API rather than concluding the action failed.
- `docker compose` reporting no services while containers are clearly running means the stack belongs to another directory; target it with `docker compose -p <project>` rather than starting a new one.
- A stack that is healthy but missing expected records after a move means the new project got fresh volumes, or the `.env` was regenerated and no longer matches the old Postgres password. Check the volume prefix and the `.env` before assuming data loss.

## 8. Report and clean up

When finishing a session that touched the running app:

- Report the exact URL and container/port used, and which compose project owns the stack.
- If the stack is running from a worktree rather than the main checkout, say so — it will not survive that session.
- Delete temporary credential or response files written outside the repository.
- Confirm the repository worktree is still clean (`git status --porcelain`) when the task was operational and was not meant to change code.
- If a real administrator was created while demo accounts remain active, point out that the documented demo credentials are still valid and offer to disable demo sign-in.
