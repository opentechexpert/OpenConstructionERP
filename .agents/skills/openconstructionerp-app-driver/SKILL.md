---
name: openconstructionerp-app-driver
description: Operate and verify the OpenConstructionERP application through Docker and browser automation. Use for starting or discovering the app, login, navigation, UI workflows, smoke tests, and end-to-end troubleshooting.
compatibility: Requires Docker, curl, and a browser automation interface when performing UI workflows. Credentials must be supplied by the user or an approved environment.
metadata:
  author: opentechexpert
  version: "1.1"
---

# OpenConstructionERP app driver

Use this skill when an agent needs to operate the running OpenConstructionERP application rather than only edit source code. The browser automation interface may be a browser tool, Playwright, WebDriver, a browser canvas, or another harness-specific adapter.

## Operating principles

- Prefer the already-running application over starting a second copy.
- Never print, commit, or expose passwords, tokens, cookies, or session headers.
- Do not invent credentials. Ask the user for credentials or use credentials explicitly provided through the environment.
- Treat destructive actions as requiring confirmation unless the user explicitly requested them.
- Verify every important action through the UI or an application/API health check; do not claim success from a click alone.
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

A healthy response reports `status: healthy` plus `version`, `modules_loaded`, `database: ok`, and `schema_matches_models: true`. Treat `schema_matches_models: false` or a non-`ok` database as a blocker and report it instead of proceeding with UI work.

### Containers owned by another worktree or session

`docker ps` may reveal a healthy stack started by a *different* worktree of the same repository. The compose project name and working directory come from wherever it was launched, so:

- `docker compose ...` run from the current worktree will **not** control that stack. Read the owning directory first:

  ```bash
  docker inspect <application-container> \
    --format '{{index .Config.Labels "com.docker.compose.project"}} {{index .Config.Labels "com.docker.compose.project_working_dir"}}'
  ```

- Run any stop/restart/rebuild from the owning directory, never from the current one.
- Ask the user before reusing a borrowed stack, and tell them the trade-off: reusing it is fastest, but the data lives in that stack's volumes and disappears if the owning session tears it down.
- Starting a competing stack on the same port will fail; offer an alternate published port instead.

### Apple Silicon (arm64) hosts

The published image is amd64 and runs under emulation on arm64. The no-clone quickstart Docker commands in `docs/getting-started.md` do not work on Apple Silicon. Clone the repository and use the arm64 path instead:

```bash
make quickstart-arm64   # uses docker-compose.arm64.yml
```

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

Element references (`e1`, `e8`, …) are invalidated by any navigation, route change, or form submission. Re-read the page to obtain fresh references before every `type`/`click` that follows a navigation, otherwise the action fails with a stale-reference error.

### Routes behave differently depending on session state

- `/register` and `/users` redirect to `/dashboard` while a session is active. To reach the public registration form, log out or wait for the session to expire, then use the **Create account** link on `/login`.
- An expired session redirects to `/login?next=<original-path>` and resumes there after login.
- A brand-new real admin lands on the 6-step setup wizard at `/onboarding`, not `/dashboard`. This is correct behaviour, not a failure — the demo accounts skip it because they are pre-onboarded. Do not click through the wizard on the user's behalf; its choices (language, modules, cost data) are theirs to make.

## 3. Authenticate safely

Determine whether the page is a login screen before entering anything. If credentials are not already available from an approved environment variable or explicit user instruction, ask the user for them.

- Enter credentials only into the application’s visible login fields.
- Do not echo the values in tool arguments, logs, summaries, screenshots, or commits when the tool surface would expose them.
- Submit the form, then inspect the page to verify that the authenticated application shell, dashboard, or requested destination is present.
- If login fails, report the visible error and stop; do not repeatedly guess credentials.
- If MFA, SSO, CAPTCHA, or an approval prompt appears, pause and ask the user to complete it.

### Demo accounts

Seeded demo accounts use the `@openconstructionerp.com` domain and their credentials are published in the repository documentation, so they are not secrets and may appear in commands. The login page also exposes a one-click demo sign-in button, which is the fastest way to smoke-test without typing a password.

Demo accounts are not a substitute for a real account. When the user asks for a "real" user, do not hand them a demo login.

### Handling a generated password

If the user asks you to generate a password, keep it out of tool arguments and logs:

```bash
umask 077 && python3 -c "import secrets,string; \
  print(''.join(secrets.choice(string.ascii_letters+string.digits+'+/') for _ in range(20)))" > /tmp/pw.txt
```

Pass it to the API by reading the file *inside* the request script, never by interpolating it into a command line. Reveal it to the user only if they explicitly asked for a one-time display, tell them to change it immediately, and delete the file when done:

```bash
shred -u /tmp/pw.txt 2>/dev/null || rm -f /tmp/pw.txt
```

## 4. API access and user administration

### Endpoint layout

Auth endpoints are nested **under the users module**, not at a top-level `/auth`. Trailing slashes are required.

```text
POST /api/v1/users/auth/login/
POST /api/v1/users/auth/register/
POST /api/v1/users/auth/refresh/
POST /api/v1/users/auth/demo-login/
GET  /api/v1/users/me/
GET  /api/v1/users/            # admin only
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
- Every later registrant defaults to `viewer` (`OE_DEFAULT_REGISTRATION_ROLE`); `admin` is never grantable through self-registration.

So the supported way to create a real admin is the ordinary registration endpoint or the `/register` form — not direct database edits. Confirm no real admin exists first, since the promotion only applies to the first one.

Password policy (`backend/app/modules/users/schemas.py`): at least one letter and one digit, not in the common-password blacklist, minimum 8 characters for public registration and 12 for the admin-created-user endpoint.

### Verifying a created user

A `201` response alone is not proof the account is usable. Confirm all of:

1. `POST /api/v1/users/auth/login/` returns a token.
2. `GET /api/v1/users/me/` reports the expected `role` and `is_active: true`.
3. For an admin, an admin-only endpoint such as `GET /api/v1/users/` returns `200` rather than `403`.
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

## 6. Smoke-test checklist

For a basic availability check, verify:

- The application container is running and healthy.
- The published root URL responds successfully.
- `/api/health` reports `status: healthy`, `database: ok`, and `schema_matches_models: true`.
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
- A `404` from an API call is usually a wrong path, not a missing feature — check the nesting and the trailing slash before concluding the endpoint does not exist.
- A stale-element-reference error is a browser-harness issue, not an application bug; re-read the page and retry once.
- `docker compose` reporting no services while containers are clearly running means the stack belongs to another directory; find its `project_working_dir` instead of starting a new one.

## 8. Report and clean up

When finishing a session that touched the running app:

- Report the exact URL and container/port used, and whether the stack was started by you or borrowed from another session.
- If the stack was borrowed, warn the user that any data created lives in that stack's volumes.
- Delete temporary credential or response files written outside the repository.
- Confirm the repository worktree is still clean (`git status --porcelain`) when the task was operational and was not meant to change code.
- If a real administrator was created while demo accounts remain active, point out that the documented demo credentials are still valid and offer to disable demo sign-in.
