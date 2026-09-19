---
name: openconstructionerp-app-driver
description: Operate and verify the OpenConstructionERP application through Docker and browser automation. Use for starting or discovering the app, login, navigation, UI workflows, smoke tests, and end-to-end troubleshooting.
compatibility: Requires Docker, curl, and a browser automation interface when performing UI workflows. Credentials must be supplied by the user or an approved environment.
metadata:
  author: opentechexpert
  version: "1.0"
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

Check readiness using the backend health contract first, then confirm the frontend separately:

```bash
curl --fail --silent --show-error --max-time 10 http://localhost:<published-port>/api/health
curl --fail --silent --show-error --max-time 10 http://localhost:<published-port>/
```

`GET /api/health` is the readiness contract; a `/` response only confirms the static frontend is being served, not that the backend/API is up.

If the application container is absent, inspect the available compose files and use the project’s documented quickstart or production command, for example `docker-compose.quickstart.yml` (or `docker-compose.quickstart.image.yml`) for a local quickstart, or `docker-compose.prod.yml` for a production-style stack. Before starting containers, check for an existing stack and avoid disrupting unrelated containers:

```bash
docker compose ps
```

Only start containers when the user asked to start the application or no suitable application container is running, and use the documented quickstart or production compose file above rather than the bare root `docker-compose.yml` (which brings up supporting services without the application itself), for example:

```bash
docker compose -f docker-compose.quickstart.yml up -d
```

After startup, wait for the container health status and re-check `/api/health` and `/`.

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

## 3. Authenticate safely

Determine whether the page is a login screen before entering anything. If credentials are not already available from an approved environment variable or explicit user instruction, ask the user for them.

- Enter credentials only into the application’s visible login fields.
- Do not echo the values in tool arguments, logs, summaries, screenshots, or commits when the tool surface would expose them.
- Submit the form, then inspect the page to verify that the authenticated application shell, dashboard, or requested destination is present.
- If login fails, report the visible error and stop; do not repeatedly guess credentials.
- If MFA, SSO, CAPTCHA, or an approval prompt appears, pause and ask the user to complete it.

## 4. Execute app workflows

Before changing data, identify whether the requested workflow is read-only or mutating. For a mutating workflow:

1. Navigate to the requested module through visible UI controls.
2. Confirm the destination page and relevant record/context.
3. Fill only the requested fields.
4. Review the form state before submitting.
5. Submit once and wait for the application response.
6. Verify the success notification, updated record, or resulting page.

For read-only investigation, prefer navigation and filtering over direct database manipulation. Use application APIs only when the user asks for API-level testing or the UI cannot expose the needed diagnostic.

## 5. Smoke-test checklist

For a basic availability check, verify:

- The application container is running and healthy.
- The published root URL responds successfully.
- The login page renders without a server error.
- Static assets load and the page has no obvious fatal error.

For an authenticated smoke test, additionally verify:

- Login completes.
- The main navigation or dashboard renders.
- One representative read-only page opens.
- Logout returns to the login page, unless the user asked to preserve the session.

Report the exact URL, container/port discovered, workflow steps performed, and any blocker. Do not report a workflow as complete without a post-action verification.

## 6. Troubleshooting

Use these checks in order:

```bash
docker ps
docker compose ps
curl --fail --silent --show-error --max-time 10 http://localhost:<published-port>/api/health
curl --fail --silent --show-error --max-time 10 http://localhost:<published-port>/
```

Only inspect container logs if you can redact secrets first: startup logs may print a one-time, auto-generated demo/seed password (see the login line emitted when no seed password is pinned via environment variable). Filter it out before viewing, for example:

```bash
docker logs --tail 100 <application-container> | grep -vi 'password\|Login:'
```

If you cannot confirm the filter removes all credential-bearing lines for the deployment at hand, skip the logs step and rely on the health/UI checks above instead.

Common distinctions:

- `localhost:5173` is usually a development server and is not the correct URL for a production Docker container.
- A healthy container with an unreachable host URL usually indicates a missing or different port mapping.
- A reachable login page does not by itself prove the backend/API is available; before classifying a failed login as an authentication or seed-data issue, confirm `GET /api/health` succeeds and inspect the actual login request/response (not just that the login page rendered).
- A reachable login page with a failed login, once `/api/health` is confirmed healthy, indicates an authentication or seed-data issue, not a Docker networking issue.
- A page that loads but lacks expected content may indicate frontend asset, API proxy, or backend readiness problems; inspect browser errors and container logs before changing code.
