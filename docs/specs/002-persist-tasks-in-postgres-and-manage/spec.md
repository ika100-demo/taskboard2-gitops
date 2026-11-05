---
spec_id: 002-persist-tasks-in-postgres-and-manage
title: Persist tasks in Postgres and manage them in the web UI
status: draft
priority: P1
shape: gitops-app
---

# 002 — Persist tasks in Postgres and manage them in the web UI

## Problem

Spec 001 gave taskboard2 an API to create and list tasks, but the tasks live only in the API's memory: every restart or redeploy wipes them (AC-001.12), and the API cannot run more than one instance without each one holding its own list. The only way to use the system is to send HTTP requests by hand, because the taskboard2 web app still shows only its template page. Now that tasks are worth keeping, they should go into a database, and people should be able to see and add tasks in a browser.

## Stories

As a user of taskboard2, I want the tasks I created to still be there after the API restarts or is redeployed, so that I can rely on the board as my record of work.
**Repos:** taskboard2-api, taskboard2-gitops

As a user of taskboard2, I want to open the web app and see all tasks with their due dates, so that I don't have to call the API by hand.
**Repos:** taskboard2-web, taskboard2-gitops

As a user of taskboard2, I want to add a task with a title and an optional due date from the web app, so that I can write down work where I read it.
**Repos:** taskboard2-web, taskboard2-api

## Acceptance criteria

Persistence (the `POST /tasks` and `GET /tasks` contract of spec 001 stays as it is; only where the tasks are kept changes):

- **AC-002.1** Given tasks were created through `POST /tasks`, when the API restarts or is redeployed, then `GET /tasks` returns the same tasks with the same `id`, `title` and `due_date` as returned at creation, in the same order (oldest first). This replaces AC-001.12.
- **AC-002.2** Given tasks exist from before an API restart, when a new task is created after the restart, then its `id` differs from the `id` of every existing task.
- **AC-002.3** Given the API runs with 2 replicas (allowed in any environment; dev runs 1), when a task is created through one request, then every following `GET /tasks` contains that task, whichever replica answers.
- **AC-002.4** Given the persistent store of an environment has never held a task, when a caller sends `GET /tasks`, then the response is `200` with `[]`. Tasks that only existed in the memory of the previous API version are not carried over.
- **AC-002.5** Given the database is unreachable, when a caller sends `GET /tasks` or `POST /tasks` with a valid body, then the response is `503` with a JSON error body, and no task is created; when the database is reachable again, both requests succeed without restarting the API.
- **AC-002.6** Given the API is backed by the database, when the checks of AC-001.1 to AC-001.11 are run against it, then they all still pass (same status codes, fields, validation rules and error bodies).

Web UI:

- **AC-002.7** Given tasks exist, including tasks created directly through the API, when a user opens the web app's root URL, then the page lists every task returned by `GET /tasks`, oldest first, each showing its title and, if it has one, its due date as `YYYY-MM-DD`; a task without a due date shows no date.
- **AC-002.8** Given no task exists, when a user opens the web app's root URL, then the page shows the text "No tasks yet." instead of a list.
- **AC-002.9** Given the web app's root URL is open, when the user enters a title (and optionally picks a due date) in the create form and submits it, then the new task appears at the end of the list without a manual page reload, the form is cleared, and `GET /tasks` on the API contains the task.
- **AC-002.10** Given the create form, when the user submits a title that is empty, only whitespace or longer than 200 characters, then the form shows an error message next to the title field, keeps what the user typed, and no task is added (the API's rules of AC-001.7 and AC-001.8 apply; the web app adds no rules of its own).
- **AC-002.11** Given the API is unreachable or answers `503`, when a user opens the root URL, then the page still loads and shows "Tasks could not be loaded. Please try again later." instead of the list; when the user submits the create form in that state, then the form shows "The task could not be saved. Please try again later." and keeps what the user typed.
- **AC-002.12** Given only the web app is exposed outside the environment and the API is reachable only inside it, when a user lists and creates tasks in the web app, then both work: the user's browser never has to reach the API directly.

## Non-goals

- Migrating tasks held in memory by the running API version: they are lost once at the switch to the database (AC-002.4).
- Any change to the API contract of spec 001: no new endpoints, fields or validation rules.
- Editing, completing or deleting tasks, in the API or the web UI.
- Filtering, sorting options, search or pagination of the list.
- Users, login or per-user task lists: every visitor of the web app sees and can create every task.
- Validation rules in the web app beyond showing what the API answers.
- Live updates: tasks created elsewhere appear in the web app on the next page load, not automatically.
- Backups, restore and retention of the database (a separate concern of the platform's `backup:` settings).

## Open questions

- ~~Confirm: tasks currently held in memory by the running API are not migrated and are lost once when the database is switched on.~~ Answered: yes, not migrated; the one-time loss is accepted; see AC-002.4 and Non-goals.
- ~~Confirm: the web app calls the API from its server side inside the environment, so the API does not need to be reachable from the browser and needs no cross-origin setup.~~ Answered: yes, server-side; see AC-002.12.
- ~~Confirm: the web list is ordered oldest first like the API, shows title and due date as `YYYY-MM-DD` (nothing for no due date), and uses the texts "No tasks yet.", "Tasks could not be loaded. Please try again later." and "The task could not be saved. Please try again later."~~ Answered: yes; see AC-002.7, AC-002.8, AC-002.11.
- ~~Now that tasks are shared through the database, should the API run with more than one replica, at least in staging and prod?~~ Answered: more than one replica is allowed and the API must work correctly with 2 replicas; dev stays at 1 replica; see AC-002.3.

## Changelog

- 2026-11-05 created
- 2026-11-05 amended: answered open questions
