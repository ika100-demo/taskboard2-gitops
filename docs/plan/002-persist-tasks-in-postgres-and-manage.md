---
plan_id: 002-persist-tasks-in-postgres-and-manage
spec: 002-persist-tasks-in-postgres-and-manage
feature: Persist tasks of taskboard2-api in Postgres and list and create them in the taskboard2-web UI
gitops_app: ika100-demo/taskboard2-gitops
status: completed
repos:
- id: taskboard2-gitops
  shape: gitops-app
  summary: Declare the postgres resource, wire it into taskboard2-api and point taskboard2-web at the
    API in-cluster
  acs:
  - AC-002.1
  - AC-002.3
  - AC-002.4
  - AC-002.12
  gitops:
  - resource: postgres
  - uses: postgres
    component: taskboard2-api
  - expose: taskboard2-api
    host: taskboard2-api
  - expose: taskboard2-web
    host: taskboard2-web
  - env:
      TASKBOARD2_API_URL: http://taskboard2-api
    component: taskboard2-web
  arguments: |
    Smoke checks: none of its own; verified through the smoke runs of both component repos after Argo reconciles
    (see "gitops-app PR" in the system plan). Both components stay exposed in dev; dev keeps replicas: 1.
  depends_on: []
  done: true
- id: taskboard2-api
  shape: service-python
  summary: Persist tasks in Postgres behind the existing TaskStore, answering 503 when the database is
    down
  acs:
  - AC-002.1
  - AC-002.2
  - AC-002.3
  - AC-002.4
  - AC-002.5
  - AC-002.6
  arguments: |
    Smoke checks: mark AC-002.6 as a smoke criterion: the existing smoke checks of AC-001.1, AC-001.2, AC-001.3,
    AC-001.7 and AC-001.10 keep running against the public URL (dev: http://taskboard2-api.taskboard2-dev.localhost:8088)
    and now go through Postgres. AC-002.1, AC-002.2, AC-002.3, AC-002.4 and AC-002.5 are in-process tests only
    (they need a restart, two app instances, an empty database or a database that is down); the dev store is
    shared and never empty, so smoke tests keep finding their own task by id.
    Implementation: swap InMemoryTaskStore for a PostgresTaskStore in create_app() (spec 001 design.md planned this);
    the router and schemas stay. Config from DATABASE_URL (set by the platform, ADR-057); without DATABASE_URL the app
    keeps the in-memory store so the existing in-process tests and local runs work. Driver: psycopg 3 with
    psycopg_pool (add through the repo's devbox/uv recipe, lockfile updated). Schema and ordering under "Contract".
    Tests against a real Postgres (testcontainers in `devbox run test`; GitHub runners have Docker), with a fresh
    database per test; a database that is down is simulated by stopping the container or using an unreachable port.
  depends_on:
  - taskboard2-gitops
  done: true
- id: taskboard2-web
  shape: web-nextjs
  summary: List tasks and create them with a form on the root page, calling taskboard2-api server-side
  acs:
  - AC-002.7
  - AC-002.8
  - AC-002.9
  - AC-002.10
  - AC-002.11
  - AC-002.12
  arguments: |
    Smoke checks: mark AC-002.7, AC-002.9, AC-002.10 and AC-002.12 as smoke criteria (`@smoke` Playwright tests,
    `SMOKE_BASE_URL` = the web app's public URL, dev: http://taskboard2-web.taskboard2-dev.localhost:8088). They use
    only the web URL: create a task through the form with a unique title (and a due date), see it at the end of the
    list with its date as YYYY-MM-DD, reload and still see it (the server-side list comes from GET /tasks); submit an
    empty title and see the field error with no task added; assert that no browser request goes to a host other
    than the web app's (AC-002.12). Never assume an empty list (dev is shared). AC-002.8 and AC-002.11 are
    in-process / local e2e only (they need an empty or unreachable API): Vitest with a mocked fetch, and Playwright
    e2e against a small stub API started by the e2e setup.
    Replaces the template home page (feature cards) with the task page; keep layout, header, footer and the
    /api/health, /api/ready, /api/metrics routes. Document TASKBOARD2_API_URL in docs/env-vars.md (server-only,
    never NEXT_PUBLIC_). The page is dynamic (no static prerender, no fetch cache).
  depends_on:
  - taskboard2-gitops
  done: true
gitops_pin: []
---

## taskboard2-gitops — Declare the postgres resource, wire it into taskboard2-api and point taskboard2-web at the API in-cluster

**Shape:** `gitops-app` · **Criteria:** AC-002.1, AC-002.3, AC-002.4, AC-002.12 · **Depends on:** []

The gitops part comes first because both component repos need what it wires at deploy time: the API's dev
Deployment needs `DATABASE_URL` (and `PG*`) from the CloudNativePG cluster before a Postgres-backed image rolls
out, and the web app needs `TASKBOARD2_API_URL` before its new page can list anything. Both are real
dependencies, not ordering for its own sake: an API image without its database would answer every task request
with `503`.

Operations, in order (`/system:ship` runs them with `cplat resource add` and `cplat component set`):

1. `resource: postgres` declares the system's database (CloudNativePG cluster, connection Secret
   `taskboard2-postgres-app`). A new, empty database: tasks held in memory by the running API are not migrated
   (AC-002.4).
2. `uses: postgres` on taskboard2-api: it gets `DATABASE_URL`, `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`,
   `PGPASSWORD`. Every replica shares the same database (AC-002.1, AC-002.3).
3. `expose` taskboard2-api and taskboard2-web on their current hosts: both stay published through the Gateway in
   dev, unchanged. The API stays exposed for its own smoke tests and direct callers; the web app does not rely on
   it (AC-002.12).
4. `env TASKBOARD2_API_URL=http://taskboard2-api` on taskboard2-web: the API's in-cluster Service (same namespace,
   port 80 to the container's 8080). The web app calls the API only from its server side (AC-002.12).

Replicas: dev stays at `replicas: 1`; AC-002.3 allows 2 in any environment, and the API is built and tested for
2, but there is no staging or prod instance yet. Scaling is a separate change of `instances.<env>.replicas`.

## taskboard2-api — Persist tasks in Postgres behind the existing TaskStore, answering 503 when the database is down

**Shape:** `service-python` · **Criteria:** AC-002.1 to AC-002.6 · **Depends on:** [taskboard2-gitops]

Spec 001 put the store behind a `TaskStore` protocol created in `create_app()`; this repo adds a
`PostgresTaskStore` and selects it when `DATABASE_URL` is set. The HTTP contract of spec 001 does not change except
for the new `503` (AC-002.5). The repo owns:

- **AC-002.1, AC-002.2:** tasks live in the `tasks` table; `id` stays a UUID4 generated by the service (unique
  across restarts by construction, and the column is the primary key); `GET /tasks` orders by a database sequence,
  never by wall-clock time.
- **AC-002.3:** no state in the process: every request reads and writes the database, so any replica answers the
  same. The schema bootstrap is safe when two replicas start at once.
- **AC-002.4:** an empty table gives `200 []`; no import of in-memory data.
- **AC-002.5:** database errors become `503` with the JSON body below, the transaction is rolled back (no task
  created), and the pool reconnects on the next request without a restart. The app starts even when the database
  is unreachable, and `/ready` does not check the database (an unready pod would make the Gateway answer with a
  non-JSON `503` and hide the contract's body).
- **AC-002.6:** the acceptance tests of AC-001.1 to AC-001.11 run against the Postgres store as well (parametrize the
  store fixture). AC-001.12 is replaced by AC-002.1; its test changes accordingly.

Test concerns: restart = a second `create_app()` on the same database; 2 replicas = two app instances on the same
database, create on one, list on the other; database down = stop the container (or point at a closed port),
expect `503` with `detail` and no row, then bring it back and expect `200`/`201` on the same app instance.

## taskboard2-web — List tasks and create them with a form on the root page, calling taskboard2-api server-side

**Shape:** `web-nextjs` · **Criteria:** AC-002.7 to AC-002.12 · **Depends on:** [taskboard2-gitops]

The web app builds against the contract below in parallel with the API: it does not need the API's code, and the
API's HTTP contract of spec 001 is unchanged. It owns:

- **AC-002.7, AC-002.8:** the root page (`app/page.tsx`, a Server Component) calls `GET /tasks` on the server and
  renders the list in the order the API returns, title and `due_date` as given (`YYYY-MM-DD`, nothing when `null`);
  `[]` shows "No tasks yet.".
- **AC-002.9:** a client form component (title text field, optional `<input type="date">`) submits through a Server
  Action that calls `POST /tasks`; on `201` the returned task is appended to the list without a page reload and the
  form is cleared.
- **AC-002.10:** the web app adds no validation: no `required`, `minLength`, `maxLength` or `pattern` on the title
  field, no trimming. It sends what the user typed and, on `422`, shows an error next to the field named last in
  each `detail[].loc` (`title`, or `due_date` next to the date field), keeps the inputs and adds nothing. The
  message text is the web app's own fixed text per field (for title: "Please enter a title of 1 to 200
  characters."), never the API's `msg`.
- **AC-002.11:** any failure of `GET /tasks` (see Errors and Timeouts) renders the page with "Tasks could not be
  loaded. Please try again later." instead of the list (the page itself returns `200`); a failed `POST /tasks` shows
  "The task could not be saved. Please try again later." and keeps the inputs.
- **AC-002.12:** the browser only ever talks to the web app (page and Server Action); `TASKBOARD2_API_URL` is read on
  the server only.

## Contract

The API's HTTP contract is the one of spec 001 (system plan `docs/plan/001-create-and-list-tasks.md`, `## Contract`),
unchanged in fields, validation and `201`/`200`/`422` behaviour; this section restates what the web app needs and
adds the database-related parts.

### Base URL and configuration

- taskboard2-api: in-cluster `http://taskboard2-api` (Service port 80); dev public URL
  `http://taskboard2-api.taskboard2-dev.localhost:8088`. Reads `DATABASE_URL` (libpq URI, set by the platform from
  the `postgres` resource). Without `DATABASE_URL` it uses the in-memory store (tests and local runs only).
- taskboard2-web: reads `TASKBOARD2_API_URL` (no trailing slash; dev value `http://taskboard2-api`) on the server
  only. When it is unset, every API call counts as failed (AC-002.11 texts), and the app still starts and passes
  its probes.

### Task object

Exactly three fields: `id` (string, UUID4 canonical form, generated by the API, unique across restarts and
replicas), `title` (string, trimmed, 1 to 200 characters), `due_date` (`"YYYY-MM-DD"` or `null`, always present).

### `GET /tasks`

- `200` with a JSON array of task objects, oldest first (creation order), every task in the database; `[]` when
  empty.
- `503` when the database is unreachable or fails (body under Errors).

### `POST /tasks`

Request: JSON object, `Content-Type: application/json`, `title` (required string) and `due_date` (optional,
`"YYYY-MM-DD"` or `null`); the web app sends `due_date` as `null` when the date field is empty, and `title` exactly
as typed.

- `201` with the created task object.
- `422` on invalid input, validated before the database is touched (so an invalid body gives `422` even when the
  database is down).
- `503` when the database is unreachable or fails; no task is stored.

### Storage (taskboard2-api only)

```sql
CREATE TABLE IF NOT EXISTS tasks (
  seq      BIGINT GENERATED ALWAYS AS IDENTITY UNIQUE,  -- creation order
  id       UUID PRIMARY KEY,
  title    VARCHAR(200) NOT NULL,
  due_date DATE NULL
);
```

- `GET /tasks` is `SELECT id, title, due_date FROM tasks ORDER BY seq`.
- The schema is created by the API itself, idempotently, under `pg_advisory_lock` so two replicas starting at once
  do not race; it runs on the first successful database connection, not as a hard startup requirement.

### Web page (taskboard2-web)

- `GET /` renders: a list (`<ul>`/`<li>`, one item per task, title then due date) or "No tasks yet." or "Tasks could
  not be loaded. Please try again later."; and the create form (title field labelled "Title", date field labelled
  "Due date", submit button "Add task").
- Field errors from a `422` are shown next to their field; the generic save error of AC-002.11 is shown above the
  submit button.

### Errors

- `422` (API): the spec 001 format `{"detail": [{"loc": [...], "msg": str, "type": str}]}`; callers depend only on the
  status and the last element of each `loc` (`"title"` or `"due_date"`; `["body"]` or `["body", <pos>]` for a body
  that is not a JSON object).
- `503` (API, new): `Content-Type: application/json`, body `{"detail": "Task store unavailable"}`. Callers depend on
  the status only, not the text. Raised for any connection, pool-timeout or query error of the database.
- Any other status, a non-JSON body, a body that does not match the shapes above, a timeout or a connection error is
  a failed call. The web app treats a failed `GET /tasks` like `503` (load error text) and a failed `POST /tasks`
  like `503` (save error text, inputs kept); on a `422` whose `loc` names no known field, it shows the save error
  text. It never assumes a failed `POST` did not create the task; the next page load shows the truth.
- The web app logs failed API calls server-side (status, duration) and never forwards API bodies to the browser.

### Timeouts

- Web to API: 5 s per request (`AbortSignal.timeout(5000)`), for both `GET /tasks` and `POST /tasks`; no retry. A
  timed-out or refused call is a failed call (Errors). A retry by the user after a timed-out `POST` may create a
  duplicate, which AC-001.6 allows.
- API to database: connect timeout 2 s, pool checkout timeout 2 s, `statement_timeout` 3 s, so the API answers
  `503` well inside the web app's 5 s. The pool drops broken connections and reconnects on the next request
  (AC-002.5), with no restart and no background retry loop.
- When the API is down or restarting, the Gateway or the in-cluster Service refuses the connection or answers
  `502`/`503`: the web app shows its error texts and keeps serving its page and probes. When the database is down,
  the API stays ready and answers `503` with the JSON body.
- Smoke tests: 10 s per request (API, as in spec 001); Playwright's default timeouts (web).

## gitops-app PR

One gitops-app PR with the five operations above, merged **before** the two component PRs ship (`depends_on`). It
adds the CloudNativePG cluster, wires `DATABASE_URL`/`PG*` into taskboard2-api and `TASKBOARD2_API_URL` into
taskboard2-web, and keeps both HTTPRoutes. No pins (`gitops_pin: []`): both components run only in `dev`, which
tracks `main`, so each new image rolls out when its PR merges. Staging and prod have no instances; adding them is a
separate `/instance:promote` decision.

After Argo reconciles dev, verify:
- the `taskboard2-postgres` cluster is healthy and the Secret `taskboard2-postgres-app` exists;
- the taskboard2-api Deployment has `DATABASE_URL` from that Secret and still `replicas: 1`; the taskboard2-web
  Deployment has `TASKBOARD2_API_URL=http://taskboard2-api`; both HTTPRoutes are attached;
- after the API PR merges: `SMOKE_BASE_URL=http://taskboard2-api.taskboard2-dev.localhost:8088 devbox run smoke` in
  taskboard2-api passes; then restart the API pod (`kubectl rollout restart`) and check that `GET /tasks` still
  lists the tasks the smoke run created (AC-002.1 on the real cluster);
- after the web PR merges: `SMOKE_BASE_URL=http://taskboard2-web.taskboard2-dev.localhost:8088 devbox run smoke` in
  taskboard2-web passes.
