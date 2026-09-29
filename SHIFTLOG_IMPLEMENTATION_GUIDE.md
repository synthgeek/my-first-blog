# DRISHTI Shiftlog: Step-by-Step Implementation Guide

Companion to the "DRISHTI ShiftLog: Claude Code Generation Prompt" spec. This guide contains steps, commands and checks only, no code. It starts from the current state of the `mysite` project and builds Shiftlog one commit-sized slice at a time.

---

## 1. Findings that shape the plan

Your real `core` models differ from the spec in ways that affect Shiftlog's design.

| Spec says | Actual repo | Consequence |
|---|---|---|
| `System.area` is a FK | `System.area` is **many-to-many** | Shiftlog must not assume one area per system. |
| `Equipment.system` is a FK | **Many-to-many** | Same. |
| `Surveillance.equipment` is a FK | **Many-to-many** | An execution targets one equipment out of the listed ones (see Decision 3). |
| `Parameter.system` is required (implied) | **Nullable** (Reactor Power has none) | `ParameterReading.system` must allow null. |
| Surveillance frequency (unspecified) | Six values, including `SHIFT` | "Due" logic cannot be date-only. Reference data uses only DAILY and WEEKLY. |
| `Surveillance.parameter` (implied optional) | **Required FK**; `clean()` forces area and system to match the parameter's | `SurveillanceExecution.parameter` can be non-null and validated against the surveillance. |

Other repo facts:

- `settings.py` picks the database with a live `DB_ENGINE` switch that has a real SQLite branch. `.env` says `postgresql`. The "no silent fallback" rule is enforced with a system check, not by editing `DATABASES`.
- `home.html` defines the `extra_css`, `extra_js`, `title` and `page_title` blocks, and its `content` block is empty. A child that overrides `extra_css` or `extra_js` **replaces** the parent's, so `dashboard.html` must call `{{ block.super }}` in both. `{% load %}` is not inherited, so `dashboard.html` needs its own `{% load static %}`.
- `home` is used by login, registration, the header brand link, change-password pages and about 15 tests. Keep the URL name `home`.
- The reference `db.sqlite3` has no auth groups. Both existing users are superusers, and user `barc` has a blank `employee_id` that will fail admin validation when saved.
- The zip contains no `.git` and no `.gitignore`.
- `settings.py` contains commented-out credentials (a mail app password and a DB password). Rotate them and delete them from the file.
- `UserAuthentication` has both a `views.py` module and a `views/` package. Avoid repeating that clash in Shiftlog.

---

## 2. Decisions and how they are applied

### Decision 1: Who advances `core.Surveillance.last_date` and `next_date`

One Shiftlog service records the execution and, in the same transaction, updates the surveillance's two dates. This is the only place Shiftlog writes to `core`.

- `last_date` becomes the execution's local date. `next_date` becomes that date plus one frequency interval, using `python-dateutil` (already in `requirements.txt`).
- The execution row snapshots the `next_date` it satisfied (its `due_date`), so history survives the change.
- Executions are immutable once recorded, and the admin is read-only for them, because deleting one would not roll the schedule back.
- `SHIFT` frequency (none in your data) counts as done if an execution exists in the current shift, and only `last_date` is touched.

### Decision 2: "Long Pending"

A **lapse** is one full missed frequency cycle after `next_date`. A surveillance is Long Pending at **10 or more lapses**, held in one named setting (`SHIFTLOG_LONG_PENDING_LAPSES = 10`). The instruction said both "more than ten" and "N>=10"; this guide uses N >= 10. Reference data will not trigger it: daily items are about 3 days overdue and weekly items are less than one cycle overdue.

### Decision 3: Multi-equipment surveillance

Deferred. All 22 reference surveillances have exactly one equipment. The service refuses, with a clear message, any surveillance that does not have exactly one, and the dashboard shows "needs attention" instead of crashing. The uniqueness rule is `(surveillance, equipment, due_date)`, so going multi-equipment later needs no constraint change.

### Decision 4: Calculated parameters

The SE enters Reactor Power and AF CF manually. The only reading source is `SE_MANUAL`. The entry form ignores `param_type`. With no reading, the tile says "No reading yet". `CALCULATED` and `SYSTEM` sources are not implemented until a formula or feed exists.

### Decision 5: Trips & Shutdowns

One tile, no `0 / 0` split. It is entered and shown as a single plain text-style field, and the stored value stays numeric, as `core` defines it. A visible "data note" on the tile flags the mismatch. The same note goes in the app README and the final report.

### Decision 6: Landing page

`home.html` stays the shell. `Shiftlog/templates/Shiftlog/dashboard.html` extends it and fills its `content` block. The `home` view (Python only, not `home.html`, `home.css` or `home.js`) delegates to the Shiftlog dashboard view. That gives `UserAuthentication` a soft dependency on `Shiftlog`, which is accepted to avoid rewriting every `home` reference.

---

## 3. Assumptions to confirm

The plan proceeds on these unless you say otherwise.

- Only the current responsible SE records surveillance executions.
- Superusers get no implicit edit rights on operational logs. Create separate non-superuser SE and CR test users.
- The first Shift row is created in the admin. Later ones are created by the handover-start service.
- CR assignments are made in the admin for now.
- If free text should be stored for Trips & Shutdowns, decide before Milestone 4, because it changes the first migration.
- Surveillance result choices (for example satisfactory / unsatisfactory) need your domain wording.

---

## 4. Working rules for every milestone

- Work on branch `feature/shiftlog`.
- Before each commit run:
  - `python manage.py check`
  - `python manage.py makemigrations --check --dry-run`
  - `python manage.py test`
- Never commit red. After the first run, add `--keepdb` to `test` to save time.
- Commit migrations together with their models.
- Never commit `.env`.
- Optional: tag each milestone, for example `git tag shiftlog-m3`.

---

## Milestone 0: Baseline and safety

1. If the real repo already has `.git` and `.gitignore`, skip `git init`. Otherwise run `git init`.
2. Ensure `.gitignore` covers `.env`, `db.sqlite3`, `.coverage`, `__pycache__/`, `staticfiles/` and `*.dump`. Run `git status` and confirm `.env` is not listed.
3. Commit the untouched state: `chore: baseline before shiftlog`.
4. Create the branch: `git switch -c feature/shiftlog`.
5. Delete the commented-out credentials from `settings.py`, rotate them, and commit: `chore: remove committed credentials`.
6. Create and activate a virtualenv, then run `pip install -r requirements.txt`.
7. Confirm `.env` has `DB_ENGINE=postgresql` plus `DB_NAME`, `DB_USER`, `DB_PASSWORD`, `DB_HOST` and `DB_PORT`.
8. Confirm PostgreSQL is running and the DB user has CREATEDB (Django creates `test_<DB_NAME>`).
9. Populate the dev database:
   - `python manage.py migrate`
   - `python manage.py loaddata postgres_data.json`
   - Ignore `data.json` and `sqlite_data.json`. Inspect `dsl_project_db_before_dummy_data.dump` with `pg_restore --list` before using it, since its name says pre-dummy-data.
10. Take a restore point before Milestone 5 mutates `core` dates:
    - `pg_dump -Fc -h <host> -U <user> -d <db> -f pre_shiftlog.dump`
    - Restore later with `pg_restore --clean --if-exists -d <db> pre_shiftlog.dump`
11. Record the baseline: `python manage.py check`, `python manage.py showmigrations`, `python manage.py test`. Note the test count.

---

## Milestone 1: Scaffold and Postgres guard

1. Run `python manage.py startapp Shiftlog` (matches the `UserAuthentication` naming).
2. In `Shiftlog/apps.py`, set `name = 'Shiftlog'` and `label = 'shiftlog'`.
3. Add `'Shiftlog.apps.ShiftlogConfig'` to `INSTALLED_APPS` after `core.apps.CoreConfig`. Leave `DATABASES` and `DB_ENGINE` untouched.
4. Delete the generated `views.py` and create a `views/` package (`__init__.py`, `dashboard.py`, `se_logs.py`, `cr_logs.py`, `handover.py`). Keep `models.py`.
5. Create a `tests/` package with `__init__.py`. Create `forms.py`, `permissions.py`, `selectors.py`, `services.py` and `urls.py` only when a milestone needs them.
6. Add a system check, registered from the app config's `ready()`, that raises an error unless the active connection vendor is `postgresql`.
7. Add a test for the check.

**Checks**

- `python manage.py check` passes.
- Negative test: temporarily set `DB_ENGINE=sqlite` in `.env`, confirm `check` fails with your message, then revert.
- `python manage.py test Shiftlog`

**Commit:** `feat(shiftlog): scaffold app and add postgres-only system check`

---

## Milestone 2: Skeleton dashboard, then make it the landing page

1. Add `Shiftlog/urls.py` with `app_name = 'shiftlog'` and a `dashboard` route. Include it in `mysite/urls.py` as `shiftlog/`, before the root include.
2. Build the dashboard view with the existing `login_required` from `UserAuthentication.decorators` and `never_cache`. It shows empty-state text for now.
3. Create `Shiftlog/templates/Shiftlog/dashboard.html`:
   - extends `UserAuthentication/home.html`;
   - has its own `{% load static %}`;
   - calls `{{ block.super }}` in `extra_css` and `extra_js`;
   - fills the `content` block.
4. Add one small Shiftlog stylesheet using existing tokens.
5. Add tests: anonymous redirect to login, authenticated 200, no-cache headers.
6. Run the server and confirm the header, clock, theme switch and `home.css` styling are intact.
7. **Commit A:** `feat(shiftlog): skeleton dashboard extending home.html`
8. Flip the landing page. Change only the `home` view to delegate to the Shiftlog dashboard view, importing it inside the function to avoid a circular import. Leave `home.html`, `home.css`, `home.js`, `base_generic.html` and all URL names alone.
9. Run the full suite. Login, registration and logout tests should pass unchanged. Add a Shiftlog test that `reverse('home')` renders `Shiftlog/dashboard.html`.
10. Log in through the browser and confirm you land on the dashboard.
11. **Commit B:** `feat(shiftlog): render shiftlog dashboard as the landing page`

---

## Milestone 3: Shifts, assignments, roles, on-duty header

1. Record the design in the commit body:
   - `Shift.date` is the shift's start date, so a night shift starting 23:00 on the 28th belongs to the 28th.
   - One shift per `(date, shift_type)`.
   - Statuses: `SCHEDULED`, `ACTIVE`, `CLOSED`. "Current shift" means the `ACTIVE` one, not the clock, so scheduled times never transfer responsibility.
2. Add `Shift` and `ShiftAssignment` (shift, user, role SE/CR, is-current flag, assigned-from and assigned-until). Add a partial unique constraint so a shift has at most one current SE. Allow several CRs.
3. Run `python manage.py makemigrations Shiftlog`. Open `0001_initial.py` and confirm it:
   - creates only `shiftlog_*` tables;
   - depends on `core`'s latest migration and the user model;
   - touches no `core_*` table.
4. Run `python manage.py sqlmigrate Shiftlog 0001`, then `python manage.py migrate Shiftlog`.
5. Add a data migration creating the groups `SHIFT_ENGINEER` and `CONTROL_ROOM_OPERATOR`.
6. Register both models in the admin.
7. Add selectors `get_current_shift` and `get_current_responsible_se`, and a first `permissions.py` helper checking role, assignment and shift status.
8. Show the on-duty header (shift type, times, responsible SE) on the dashboard, with a clear "no active shift" state.
9. Tests:
   - shift windows, including midnight crossing;
   - one current SE per shift;
   - selectors with no shift, one active shift and an ended shift.
10. Manual: create two non-superuser users with five-digit employee IDs in the admin and put one in each group. Create today's Shift as ACTIVE with the SE assigned, and view the dashboard as each user.

**Commits**

- `feat(shiftlog): shift and assignment models with role groups`
- `feat(shiftlog): on-duty header on dashboard`

---

## Milestone 4: Parameter readings and operational status

1. Add `ParameterReading` with direct `PROTECT` FKs to `core.Area`, `core.System` (nullable), `core.Equipment` and `core.Parameter`, plus shift, value, `recorded_at`, source (`SE_MANUAL` only) and `recorded_by`. Add an index on `(parameter, recorded_at)`. Migrate and inspect as in Milestone 3.
2. Add a parameter-definition layer in `selectors.py` for the five dashboard parameters: Reactor Power, Activity Readings, Trips & Shutdowns, AF CF, Exhaust Air Activity.
   - Resolve them through the ORM by name, disambiguated by equipment, system and area where needed.
   - Never use primary keys.
   - Fail loudly on zero or multiple matches.
3. Add a `record_parameter_reading` service that:
   - validates the area, system and equipment against the parameter's own;
   - requires an active shift and the current responsible SE;
   - accepts manual entry for CALCULATED parameters.
4. Add selectors `get_latest_parameter_reading` and `get_operational_status`. Use limits only where the parameter defines them (Activity Readings, Exhaust Air). Invent no thresholds for Reactor Power.
5. Build the `operational_status.html` partial from one reusable metric-card component.
   - Trips & Shutdowns renders as a single text-style value with the data-note marker.
   - Empty state is "No reading yet".
6. Tests:
   - a reading references the `core` records;
   - a reading keeps its value after `core.Parameter.value` changes;
   - non-SE users are refused;
   - the selector fails loudly on a missing or duplicate parameter.
7. Manual: record a few readings from `python manage.py shell` through the service, and check the tiles in light and dark mode.

**Commits**

- `feat(shiftlog): parameter reading model and service`
- `feat(shiftlog): operational status tiles`

---

## Milestone 5: Surveillance execution and status

1. Confirm the `pre_shiftlog.dump` restore point exists.
2. Add `SurveillanceExecution` with `PROTECT` FKs to area, system (nullable), equipment, parameter and surveillance, plus shift, `due_date`, execution time, executor, result and remarks. Either result counts as performed. Unique on `(surveillance, equipment, due_date)`. Migrate and inspect.
3. Add a schedule helper covering interval by frequency, next-date calculation and lapse counting. All functions take an as-of date so tests are deterministic. Add the setting `SHIFTLOG_LONG_PENDING_LAPSES = 10`.
4. Add the `record_surveillance_execution` service. It is atomic, locks the surveillance row, and:
   - enforces exactly one equipment;
   - checks parameter, area and system against the surveillance;
   - stores `due_date`;
   - advances `last_date` and `next_date`.
5. Add the summary selector with mutually exclusive buckets:
   - **Completed Today:** executions whose local date is today.
   - **Long Pending:** active, `next_date` reached, lapses >= N.
   - **Due Today:** active, `next_date` reached, lapses < N (includes overdue).
6. Build the `surveillance_status.html` partial.
7. Tests:
   - next-date maths for every frequency;
   - lapse counting at boundaries 9, 10 and 11;
   - immutability;
   - refusal for multi-equipment surveillances;
   - duplicate execution rejected;
   - bucket counts against reference-shaped data.
8. Manual: record an execution, confirm the counts move and the `core` dates advance. Restore the dump afterwards to get the original data back.

**Commit:** `feat(shiftlog): surveillance execution, schedule advancement and status`

---

## Milestone 6: SE log and signing

1. Add `SELog` (shift, author, content, status DRAFT/SIGNED, timestamps). Its form does field validation only.
2. Define "signed" as: all five dashboard parameters have a reading in this shift, and the required log fields are present. The signing service enforces this.
3. Complete `permissions.py`: can this user edit the current SE log, based on group, current assignment, shift status and log status.
4. Add current-shift routes so the user never picks a shift. Every mutating view checks permission server-side before calling a service.
5. Build the `se_log_action.html` and `signed_se_logs.html` partials and a recent-signed-logs selector. Historical logs are view-only.
6. Tests: create and update, sign only when complete, a signed log cannot be edited, the assigned SE is allowed, a view-only user is denied, a direct URL hit is denied.

**Commits**

- `feat(shiftlog): SE log with server-side permissions`
- `feat(shiftlog): signing workflow and recent logs`

---

## Milestone 7: Handover

1. Add `Handover` with outgoing and incoming shift and SE, a status of `NOT_STARTED`, `IN_PROGRESS`, `COMPLETED` or `TRANSFERRED`, and text sections (open events, pending actions, observations, management instructions, acknowledgements). Management instructions stay a text section, not a model. Only the service may change status, through an explicit allowed-transitions table.
2. Add services, each atomic and row-locked:
   - **start handover:** creates or gets the next Shift row;
   - **complete handover:** only if the SE log is signed;
   - **transfer responsibility:** flips the current-assignment flags, closes the outgoing shift and activates the incoming one.
3. Derive pending surveillance and abnormalities in the handover from selectors, not stored copies.
4. Add handover routes and the `handover.html` partial.
5. Tests:
   - the assigned SE can edit;
   - the outgoing SE is denied after transfer;
   - the incoming SE is allowed;
   - the assigned CR can edit the CR log;
   - a view-only user is denied;
   - a direct URL is denied;
   - illegal transitions;
   - a double-transfer race.

**Commits**

- `feat(shiftlog): handover model and state machine`
- `feat(shiftlog): handover views and dashboard panel`

---

## Milestone 8: CR area log

1. Add `CRAreaLog` (shift, author, optional `core.Area` and `core.System`, entry text, timestamp, status). Keep it to what the spec states, with no extra CR workflow.
2. Add its form, service, permission (assigned CR, active shift), views and the `cr_log_action.html` partial. The dashboard shows CR sections for CR users and SE sections for SEs, in the same template.
3. Add a recent-entries selector.
4. Tests for permissions, creation and rendering per role.

**Commit:** `feat(shiftlog): CR area log`

---

## Milestone 9: CSS and UI polish

1. Audit new CSS against `mysite/static/mysite/css/design/tokens.css`. Reuse existing tokens. Add one only for a genuinely new reusable concept, in both light and dark sections.
2. Check dark-mode parity, keyboard navigation, focus styles, colour contrast, and that status is never conveyed by colour alone.
3. Check narrow viewports, and confirm nothing from `UserAuthentication` was duplicated.
4. Confirm `home.css` and `home.js` are untouched: `git diff --stat main -- UserAuthentication/static`.

**Commit:** `style(shiftlog): design-token audit, dark mode and responsive pass`

---

## Milestone 10: Final verification and report

1. Run `python manage.py check`, `python manage.py makemigrations --check --dry-run`, `python manage.py migrate --plan` and the full `python manage.py test`. Compare the test count with the Milestone 0 baseline.
2. Run `sqlmigrate` for each Shiftlog migration and confirm none touches `core_*` tables.
3. Add a Shiftlog README documenting the decisions above and the known mismatches and limitations:
   - Trips & Shutdowns single value;
   - multi-equipment deferred;
   - `SHIFT` frequency handling;
   - the first-Shift bootstrap.
4. End-to-end by hand: an SE and a CR log in, record readings and an execution, sign the log, hand over, and confirm the outgoing SE is refused.
5. Produce the report the spec (section 28) asks for, including the explicit statement that `db.sqlite3` was not used as the runtime database.
6. Commit `docs(shiftlog): README and final report`, then merge with a normal review.

---

## Appendix: Command cheat sheet

| Purpose | Command |
|---|---|
| Health check | `python manage.py check` |
| Migration drift | `python manage.py makemigrations --check --dry-run` |
| Create Shiftlog migrations | `python manage.py makemigrations Shiftlog` |
| Inspect SQL | `python manage.py sqlmigrate Shiftlog 0001` |
| Apply | `python manage.py migrate Shiftlog` |
| Show plan | `python manage.py migrate --plan` |
| Tests (app) | `python manage.py test Shiftlog -v 2 --keepdb` |
| Tests (all) | `python manage.py test --keepdb` |
| Restore point | `pg_dump -Fc -h <host> -U <user> -d <db> -f pre_shiftlog.dump` |
| Restore | `pg_restore --clean --if-exists -d <db> pre_shiftlog.dump` |
| Load dev data | `python manage.py loaddata postgres_data.json` |
