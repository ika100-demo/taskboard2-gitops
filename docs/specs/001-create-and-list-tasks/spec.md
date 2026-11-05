---
spec_id: 001-create-and-list-tasks
title: Create and list tasks
status: approved
priority: P1
shape: gitops-app
---

# 001 — Create and list tasks

## Problem

The taskboard2 system has no way to record work yet: a caller of the taskboard2 API cannot add a task or see the tasks it already added. This is the first feature of the system, and every later one (completing, editing or filtering tasks) needs tasks to exist. For now the tasks only have to live as long as the service runs, so callers can try the system before any storage is set up.

## Stories

As a user of the taskboard2 API, I want to create a task with a title and, if I want, a due date, so that I can write down what I need to do.
**Repos:** taskboard2-api

As a user of the taskboard2 API, I want to list all tasks I created, so that I can see what is still to do and when it is due.
**Repos:** taskboard2-api

## Acceptance criteria

- **AC-001.1** Given the API is running, when a caller sends `POST /tasks` with the JSON body `{"title": "Buy milk"}`, then the response is `201` with a JSON object holding a non-empty `id`, `title` equal to `"Buy milk"` and `due_date` equal to `null`.
- **AC-001.2** Given the API is running, when a caller sends `POST /tasks` with `{"title": "File taxes", "due_date": "2026-12-01"}`, then the response is `201` and the returned task has `due_date` equal to `"2026-12-01"`.
- **AC-001.3** Given a task was created, when a caller sends `GET /tasks`, then the response is `200` with a JSON array that contains an object with that task's `id`, `title` and `due_date`, exactly as returned at creation.
- **AC-001.4** Given no task has been created since the service started, when a caller sends `GET /tasks`, then the response is `200` with an empty JSON array `[]`.
- **AC-001.5** Given tasks A, B and C were created in that order, when a caller sends `GET /tasks`, then they appear in the order A, B, C (oldest first).
- **AC-001.6** Given two tasks were created with the same title, when a caller lists the tasks, then both appear and their `id` values differ.
- **AC-001.7** Given the API is running, when a caller sends `POST /tasks` with no `title`, with `title` not a string, or with a `title` that is empty or only whitespace, then the response is `422` with a JSON error body that names the field `title`, and no task is added to the list.
- **AC-001.8** Given the API is running, when a caller sends `POST /tasks` with a `title` longer than 200 characters, then the response is `422` with a JSON error body that names the field `title`, and no task is added; a title of exactly 200 characters is accepted.
- **AC-001.9** Given a title with leading or trailing whitespace, such as `"  Buy milk  "`, when the task is created, then the stored and returned `title` is `"Buy milk"`.
- **AC-001.10** Given the API is running, when a caller sends `POST /tasks` with a `due_date` that is not a calendar date in the form `YYYY-MM-DD` (for example `"tomorrow"`, `"2026-02-30"` or `"2026-12-01T10:00:00Z"`), then the response is `422` with a JSON error body that names the field `due_date`, and no task is added.
- **AC-001.11** Given today is 2026-11-05, when a caller creates a task with `due_date` `"2026-01-01"` (in the past), then the task is created with that due date.
- **AC-001.12** Given tasks were created, when the service restarts, then `GET /tasks` returns an empty array `[]`.

## Non-goals

- Persistent storage: tasks are kept in memory only and are lost on restart (AC-001.12).
- Running more than one replica: the service runs as a single instance, so all callers see one task list.
- Editing, completing or deleting tasks.
- Filtering, sorting options, search or pagination of the list.
- Users, login or per-user task lists: every caller sees every task.
- Due times or time zones: a due date is a calendar date only.
- A web front end.

## Open questions

- ~~Title rules: is a title required, trimmed of surrounding whitespace and limited to 200 characters?~~ Answered: yes, required, trimmed, 1 to 200 characters after trimming; see AC-001.7, AC-001.8, AC-001.9.
- ~~Confirm: due date is a plain calendar date `YYYY-MM-DD` with no time, and dates in the past are accepted.~~ Answered: yes; see AC-001.10, AC-001.11.
- ~~Confirm: the list is ordered oldest first, by creation.~~ Answered: yes; see AC-001.5.
- ~~Confirm: losing all tasks on restart (and having separate task lists if the service ever runs with more than one replica) is acceptable for this feature.~~ Answered: yes, memory only, one replica; see AC-001.12 and Non-goals.

## Changelog

- 2026-11-05 created
- 2026-11-05 amended: answered open questions; removed smoke markers (not supported for gitops-app)
- 2026-11-05 approved
