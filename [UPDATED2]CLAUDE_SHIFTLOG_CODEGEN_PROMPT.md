# DRISHTI ShiftLog — Claude Code Generation Prompt

## 0. Objective

Modify the existing Django project to create the initial `Shiftlog` app and its dashboard/workflows.

**Authoritative source of plant/master data:** the existing project's `core` Django models in its **PostgreSQL database** the dummy has been created from that only and will not be given as prompt input. Assume it exists.

The uploaded `db.sqlite3` is only a **reference copy of core-model data/schema for understanding field names, relationships and example records**. It is **NOT the application database** and must NOT be used by the generated Shiftlog code.

Use the supplied visual references:

- `references/homeDOThtml.png` — current authenticated home.
- `references/Suggested_Dashboard.png` — target SE/CR dashboard.
- `references/a_wide_infographic_style_ui_mockup_storyboard_im.png` — workflow reference.
- `references/a_clean_multi_panel_infographic_mockup_of_a_web_ap.png` — workflow/UI reference.

Inspect the repository and its documentation before editing.

---

# 1. DATABASE RULE — NON-NEGOTIABLE

## PostgreSQL only

The existing Django project is already configured to use PostgreSQL through its environment/database configuration.

**Shiftlog must use the same PostgreSQL database connection as the existing project.**

The code must:

- import/reference `core.models` normally;
- use Django ORM relationships to `core.Area`, `core.System`, `core.Equipment`, `core.Parameter`, and `core.Surveillance`;
- create only Shiftlog-owned tables in the existing PostgreSQL database through Django migrations;
- store historical shift readings/execution records in Shiftlog tables;
- read master/definition attributes from `core` at runtime.

### NEVER do these

Do NOT:

- configure `db.sqlite3` as a database;
- add a SQLite database alias;
- create a second SQLite database for Shiftlog;
- copy/import `db.sqlite3` into PostgreSQL;
- create duplicate Shiftlog copies of Area/System/Equipment/Parameter/Surveillance;
- hard-code parameter primary-key IDs taken from the SQLite copy;
- write code that silently falls back to SQLite;
- use SQLite-specific SQL;
- run Shiftlog migrations against SQLite;
- use the uploaded SQLite database as the runtime data source.

If the PostgreSQL environment/database configuration is missing or cannot be verified, **stop and report the problem instead of falling back to `db.sqlite3`.**

Use the project's existing default Django database connection. Do not introduce a new database configuration unless the repository already requires it.

### Important distinction

```text
Uploaded db.sqlite3
    = reference/example copy ONLY

Existing PostgreSQL database
    = authoritative runtime database

core.models
    = Django model interface to plant/master data

Shiftlog models
    = historical operational/shift records
```

The final code should work when `db.sqlite3` is completely absent.

---

# 2. CORE vs SHIFTLOG BOUNDARY

`core` describes the plant/master definitions:

```text
core.Area
core.System
core.Equipment
core.Parameter
core.Surveillance
```

`Shiftlog` describes what happened during a shift:

```text
Shift
Shift assignments
SE Shift Log
CR Area Log
Parameter Reading
Surveillance Execution
Handover
Handover state/responsibility
```

Do not reproduce core master-data fields in Shiftlog unless there is a demonstrated historical/snapshot requirement.

Prefer relationships such as:

```python
area = models.ForeignKey(
    "core.Area",
    on_delete=models.PROTECT,
    related_name="shiftlog_records",
)

system = models.ForeignKey(
    "core.System",
    on_delete=models.PROTECT,
    related_name="shiftlog_records",
)

equipment = models.ForeignKey(
    "core.Equipment",
    on_delete=models.PROTECT,
    related_name="shiftlog_records",
)

parameter = models.ForeignKey(
    "core.Parameter",
    on_delete=models.PROTECT,
    related_name="shift_readings",
)
```

and:

```python
area = models.ForeignKey(
    "core.Area",
    on_delete=models.PROTECT,
    related_name="surveillance_shiftlog_records",
)

system = models.ForeignKey(
    "core.System",
    on_delete=models.PROTECT,
    related_name="surveillance_shiftlog_records",
)

equipment = models.ForeignKey(
    "core.Equipment",
    on_delete=models.PROTECT,
    related_name="surveillance_shiftlog_records",
)

surveillance = models.ForeignKey(
    "core.Surveillance",
    on_delete=models.PROTECT,
    related_name="shift_executions",
)
```

For both `ParameterReading` and `SurveillanceExecution`, the Area/System/Equipment references must point directly to the corresponding existing `core` records. Do not create Shiftlog copies of these master entities.

This means:

```text
core.Area
core.System
core.Equipment
core.Parameter
        │
        │ define the master context
        ▼
Shiftlog.ParameterReading
        │
        ├── shift
        ├── area → core.Area
        ├── system → core.System
        ├── equipment → core.Equipment
        ├── parameter → core.Parameter
        ├── value
        ├── timestamp
        ├── source
        └── recorded_by
```

Likewise:

```text
core.Area
core.System
core.Equipment
core.Surveillance
        │
        │ define the surveillance and its plant context
        ▼
Shiftlog.SurveillanceExecution
        │
        ├── shift
        ├── area → core.Area
        ├── system → core.System
        ├── equipment → core.Equipment
        ├── surveillance → core.Surveillance
        ├── execution time
        ├── executor
        ├── result/status
        └── remarks
```

Do not make second `Shiftlog.Area`, `Shiftlog.System`, `Shiftlog.Equipment`, `Shiftlog.Parameter`, or `Shiftlog.Surveillance` master tables.

---

# 3. CORE DATA REFERENCE — IMPORTANT

The supplied SQLite copy was inspected only to understand the current `core` model vocabulary and example data.

The current core model contains:

### `Parameter`

Relevant fields include:

```text
name
param_type
value                  # latest/current value in core
unit
min_opt_limit
max_opt_limit
ts_limit
area
equipment
system
is_active
```

### `Surveillance`

Relevant fields include:

```text
rule
description
frequency
result_message
last_date
next_date
area
system
equipment
parameter
is_active
```

### `Area`

```text
name
abbreviation
is_active
```

### `System`

```text
name
abbreviation
usi_number
area
is_active
```

### `Equipment`

```text
name
abbreviation
equip_type
area
system
is_active
```

Use the actual Django models in the repository as the authoritative schema. The SQLite copy is not authoritative for migrations, PKs, or runtime values.

---

# 4. DASHBOARD PARAMETER SCOPE

Do **not** implement every parameter in the dummy/core dataset.

The visual dashboard only needs the operational data represented in the supplied dashboard/workflow mockups.

The current reference dataset contains these directly relevant core parameters:

```text
Reactor Power
Activity Readings
Trips & Shutdowns
AF CF
Exhaust Air Activity
```

Their example `param_type` values in the reference copy are:

```text
Reactor Power          CALCULATED
Activity Readings     MANUAL
Trips & Shutdowns     MANUAL
AF CF                  CALCULATED
Exhaust Air Activity   MANUAL
```

Their exact database IDs in the SQLite reference file are NOT to be used in code, because PostgreSQL IDs may differ.

Resolve the actual PostgreSQL `core.Parameter` records through the Django ORM using stable identifying attributes/relationships, not copied SQLite PKs.

For example, use the parameter's actual:

```text
name
equipment
system
area
```

where needed to disambiguate.

Prefer a small, explicit parameter-definition/selector layer over scattering literal names throughout templates.

### Do not invent missing core parameters

Some workflow mockups visually show examples such as:

```text
Reactor Thermal Power
Coolant Inlet Temperature
Primary Pressure
Containment Pressure
```

If those are not present in the actual PostgreSQL `core.Parameter` data, **do not create duplicate/mock core parameters inside Shiftlog merely to reproduce the image.**

Use the actual PostgreSQL core data references as given in dummy database.

Where the visual mockup and current core data differ, preserve the visual structure but use real available core definitions.

---

# 5. HOW THE DASHBOARD USES THOSE PARAMETERS

The dashboard's operational status should be derived from Shiftlog's latest historical readings for the selected core parameters.

Conceptually:

```text
core.Parameter
       │
       │ identifies parameter
       ▼
Shiftlog.ParameterReading
       │
       │ historical readings
       ▼
latest reading for current shift/current context
       │
       ▼
dashboard operational status
```

Examples:

### Reactor Status

Can be derived from `Reactor Power`, but do not invent plant-specific status thresholds unless they are explicitly defined by existing project data/rules.

### Activity Readings

Use the historical `Activity Readings` value and the parameter's existing limits where appropriate.

### Trips & Shutdowns

The current reference core model contains a single manual `Trips & Shutdowns` parameter represented as a numeric value.

Do **not** silently invent two new core parameters (`Trips` and `Shutdowns`) just because the mockup visually displays `0 / 0`.

If the existing PostgreSQL model/data does not distinguish them, preserve the actual source semantics and flag the UI/data-model mismatch rather than fabricating data.

### Availability / Capacity Factor

The reference data contains `AF CF`.

Do not invent a second core parameter for Availability unless one exists in PostgreSQL.

If the UI displays two separate concepts, only derive/separate them if an explicit domain formula/source exists in the project. Otherwise represent the available `AF CF` meaning faithfully and document the assumption.

### Exhaust Air Radio-activities

Use `Exhaust Air Activity`.

Do not create another master parameter.

---

# 6. HISTORICAL READINGS — WHY SHIFTLOG STORES THEM

`core.Parameter.value` represents the current/latest value and is not sufficient as the historical shift record.

Shiftlog therefore needs a historical reading model, e.g.:

```text
ParameterReading
├── shift
├── area → core.Area
├── system → core.System
├── equipment → core.Equipment
├── parameter → core.Parameter
├── value
├── recorded_at
├── source
└── recorded_by
```

Possible source categories may include:

```text
SE_MANUAL
CR_MANUAL
CALCULATED
SYSTEM
```

Only implement categories that are justified by the existing application requirements.

Important:

- A reading must point to the **core Area, System, Equipment, and Parameter definitions**.
- The Area/System/Equipment references provide the explicit plant context for the reading and must point to existing `core` records.
- The reading stores the historical value.
- Do not copy the parameter name/unit/limits into every reading merely for convenience.
- At display time, obtain current definition metadata from the related core Parameter.
- Preserve historical value/timestamp even if the current core Parameter value later changes.

---

# 7. SURVEILLANCE ARCHITECTURE

The dashboard needs:

```text
Due Today
Completed Today
Long Pending
```

The actual surveillance definitions come from:

```text
core.Surveillance
```

Shiftlog records their execution.

Use:

```text
core.Area
core.System
core.Equipment
core.Surveillance
        ↓
Shiftlog.SurveillanceExecution
```

Each `SurveillanceExecution` must explicitly reference the relevant `core.Area`, `core.System`, `core.Equipment`, and `core.Surveillance` records.

Do not duplicate surveillance definitions.

The reference SQLite dataset contains surveillance definitions for various pump-pressure and valve-position checks. These are examples of the current core data and are not instructions to create duplicate records.

For the dashboard:

- query active `core.Surveillance` definitions;
- determine which are due according to their actual core scheduling fields/rules;
- use Shiftlog execution records to determine completed/pending state;
- derive summary counts from the Shiftlog execution history and core definitions.

Do not copy the sample surveillance names shown in the generated visual mockups if they do not exist in PostgreSQL.

---

# 8. CURRENT SHIFT

There are three scheduled shifts:

```text
DAY      07:00–15:00
EVENING  15:00–23:00
NIGHT    23:00–07:00 next day
```

But actual crew-change/operational responsibility is controlled by handover.

Therefore distinguish:

```text
scheduled shift time
```

from:

```text
actual operational/handover transition
```

A Shift needs concepts for:

```text
date
shift_type
scheduled_start
scheduled_end
actual start/end where required
status
SE assignment
CR assignment(s)
handover
```

Do not assume the scheduled end automatically transfers operational responsibility.

---

# 9. HANDOVER IS A STATE TRANSITION

Handover is NOT a text field.

The outgoing SE:

```text
current responsibility
        ↓
complete required readings/log
        ↓
review important matters
        ↓
identify incoming SE
        ↓
complete handover protocol
        ↓
transfer responsibility
```

After transfer:

```text
incoming SE = current responsible SE
outgoing SE = no longer current editor
```

The same general assignment principle applies to CR operators.

"Signed" means the required SE log/handover protocol has been completed with required readings/sections filled.

Define this lifecycle in the Shiftlog domain model/service layer.

Do not implement `signed=True` as a free-standing UI switch with no workflow enforcement.

---

# 10. ROLE VS SHIFT ASSIGNMENT

Role and current assignment are different concepts.

Use Django's existing User + Groups/permissions for broad role identity:

```text
SHIFT_ENGINEER
CONTROL_ROOM_OPERATOR
```

Do not create a custom User model merely to implement the initial dashboard.

Shiftlog should maintain the actual current assignment:

```text
User
  ↓
role/group

Shift
  ↓
SE assignment
  ↓
CR assignment(s)
```

Editing permission should depend on:

```text
role
+
current shift assignment
+
handover/current responsibility
+
shift/log state
```

not merely:

```python
user.groups.filter(...)
```

---

# 11. AUTHORIZATION RULE

Template conditions are NOT the security boundary.

For example:

```django
{% if can_edit_se_log %}
    <button>CREATE / EDIT SHIFT ENGINEER'S LOG</button>
{% endif %}
```

only controls presentation.

The corresponding backend endpoint must independently verify authorization.

A user must not gain access merely by manually entering an edit URL.

Use:

```text
permissions.py
```

for domain authorization and enforce it in views/services.

---

# 12. ONE SHARED DASHBOARD

Use one:

```text
Shiftlog/templates/Shiftlog/dashboard.html
```

for SE and CR.

Do not create separate SE/CR dashboard templates unless their layouts later become materially different.

Use reusable partials:

```text
Shiftlog/templates/Shiftlog/
├── dashboard.html
└── partials/
    ├── operational_status.html
    ├── surveillance_status.html
    ├── signed_se_logs.html
    ├── handover.html
    ├── management_instructions.html
    ├── se_log_action.html
    └── cr_log_action.html
```

The dashboard is a composition of shared components plus role-specific sections.

---

# 13. TEMPLATE INHERITANCE

Do NOT modify:

```text
UserAuthentication/templates/UserAuthentication/home.html
```

Do NOT modify its CSS or JS for Shiftlog.

Use:

```django
{% extends "UserAuthentication/home.html" %}
```

for:

```text
Shiftlog/templates/Shiftlog/dashboard.html
```

The hierarchy is:

```text
base_generic.html
        ↑
UserAuthentication/home.html
        ↑
Shiftlog/dashboard.html
```

`base_generic.html` remains the authenticated application shell.

Do not reproduce its:

- header
- theme switch
- clock
- user controls
- logout
- feedback
- global shell

inside Shiftlog.

---

# 14. DASHBOARD CONTENT

## SE

Use the supplied dashboard visual as the layout reference.

Include:

```text
Current shift / on-duty state

Operational Status
    Reactor status
    Activity readings
    Trips & shutdowns
    Availability/capacity-related metric from actual source data
    Exhaust-air activity

Surveillance Status
    Due today
    Completed today
    Long pending

Shift Handover
    Open events
    Pending actions
    Pending surveillance
    Abnormalities/observations
    Management instructions
    Acknowledgements

Recent signed SE logs

CREATE / EDIT SHIFT ENGINEER'S LOG

Management instructions
    currently part of handover
```

## CR

Include:

```text
Current shift / on-duty state

Operational Status
    same actual core/Shiftlog sources as SE

Surveillance Status

Recent signed SE logs

CREATE / EDIT CONTROL ROOM AREA LOG

Recent CR area log entries
```

Do not invent additional operational parameters merely to fill visual space.

---

# 15. MANAGEMENT INSTRUCTIONS

For the current implementation:

```text
Management instruction
    = text entered as part of SE handover
```

Do NOT create a separate `ManagementInstruction` model yet.

Keep the design extensible for a future ARS/RS/management role system.

---

# 16. CR AREA LOG

Create a separate Shiftlog model for the CR area log.

It must be associated with the relevant current shift and user/CR assignment.

It should support the dashboard/workflow shown in the visual reference:

```text
current shift
area/system context where applicable
observation/event/action
timestamp
author
draft/signed/final state as actually required
```

Do not invent a detailed CR workflow that has not yet been specified. Keep the initial model boundary focused on current requirements.

---

# 17. CSS OWNERSHIP

Use the existing design system:

```text
mysite/static/
└── design system
```

as the foundation.

Do not duplicate existing UserAuthentication components inside Shiftlog.

Shiftlog owns only genuinely domain-specific components:

```text
Shiftlog/static/Shiftlog/css/
├── components/
└── layout/
```

Start with the actual UI needs. Do not blindly create:

```text
dashboard-card.css
metric-card.css
status-badge.css
section-header.css
data-table.css
surveillance-summary.css
handover-panel.css
instruction-panel.css
signed-log-list.css
action-bar.css
```

Those are candidate concepts, not mandatory files.

Prefer:

```text
generic reusable visual component
        +
Shiftlog data
```

rather than a separate CSS class/file for every metric.

For example:

```text
metric-card
    → Reactor Power
    → Activity Readings
    → AF CF
```

should normally use one visual component with different data/state.

---

# 18. DESIGN SYSTEM / TOKENS

Follow the existing styling documentation.

Before adding tokens:

1. search existing tokens;
2. reuse them if possible;
3. add a new token only for a genuine reusable design concept;
4. prefer semantic aliases over a parallel vocabulary;
5. avoid turning every CSS property into a global token.

Maintain:

- token-first styling;
- documented CSS import order;
- dark-mode parity;
- accessibility;
- responsive behavior;
- existing typography;
- existing spacing/radius/shadow conventions.

Do not modify `home.css` or `home.js`.

---

# 19. PYTHON STRUCTURE

Adapt this to the project's actual conventions:

```text
Shiftlog/
├── admin.py
├── apps.py
├── models.py
├── forms.py
├── permissions.py
├── selectors.py
├── services.py
├── urls.py
├── views/
│   ├── __init__.py
│   ├── dashboard.py
│   ├── se_logs.py
│   ├── cr_logs.py
│   └── handover.py
├── templates/
│   └── Shiftlog/
│       ├── dashboard.html
│       ├── partials/
│       ├── se_logs/
│       ├── cr_logs/
│       └── handover/
├── static/
│   └── Shiftlog/
│       └── css/
│           ├── components/
│           └── layout/
└── tests/
    ├── test_models.py
    ├── test_forms.py
    ├── test_permissions.py
    ├── test_selectors.py
    ├── test_services.py
    ├── test_views.py
    └── test_dashboard.py
```

Do not create files solely because this list contains them if the existing project has a better convention.

---

# 20. FILE RESPONSIBILITIES

### `forms.py`

User-submitted data validation:

```text
required fields
field validation
range validation
cross-field validation
```

No major state-transition workflow.

### `permissions.py`

Answers:

```text
Can this user edit this current SE log?
Can this user edit this CR log?
Can this user start/complete handover?
Can this user transfer responsibility?
```

### `selectors.py`

Read/query operations:

```text
get_current_shift()
get_current_responsible_se()
get_latest_parameter_reading()
get_operational_status()
get_surveillance_summary()
get_recent_signed_se_logs()
get_dashboard_context()
```

Should not mutate database state.

### `services.py`

Business workflows/state changes:

```text
create/update SE log
create/update CR area log
record parameter reading
record surveillance execution
start handover
complete handover
transfer responsibility
sign/finalize log
```

Use transactions where a workflow changes multiple related records.

---

# 21. DATABASE / ORM IMPLEMENTATION GUARDRAILS

When defining Shiftlog models:

### Correct

```python
class ParameterReading(models.Model):
    shift = models.ForeignKey("Shift", ...)
    area = models.ForeignKey("core.Area", ...)
    system = models.ForeignKey("core.System", ...)
    equipment = models.ForeignKey("core.Equipment", ...)
    parameter = models.ForeignKey("core.Parameter", ...)
    value = models.FloatField(...)
    recorded_at = models.DateTimeField(...)
```

`SurveillanceExecution` must likewise contain direct ForeignKeys to:

```text
core.Area
core.System
core.Equipment
core.Surveillance
```

### Incorrect

```python
class ShiftlogParameter(models.Model):
    name = ...
    unit = ...
    area = ...
    equipment = ...
```

Do not recreate the core master model.

Likewise, do not copy:

```text
Area
System
Equipment
Surveillance
Parameter
```

into Shiftlog.

Reference them.

### Migration rule

Shiftlog migrations should create only Shiftlog-owned tables.

They must not recreate:

```text
core_area
core_system
core_equipment
core_parameter
core_surveillance
```

---

# 22. DO NOT HARD-CODE SQLITE IDS

The reference SQLite data may show:

```text
Reactor Power → id 67
Activity Readings → id 68
...
```

Those IDs are irrelevant to runtime.

Never write:

```python
Parameter.objects.get(pk=67)
```

because PostgreSQL may have completely different IDs.

Instead use stable domain identification through the actual core records, preferably through a selector/configuration layer.

If the project later introduces stable parameter codes, use those codes.

---

# 23. DO NOT IMPORT THE SQLITE DATA

There is no requirement to:

```text
dump SQLite
→ migrate
→ load SQLite
→ copy core data
→ seed Shiftlog
```

Do not implement any of that.

The expected architecture is:

```text
PostgreSQL
│
├── core_area
├── core_system
├── core_equipment
├── core_parameter
├── core_surveillance
│
└── Shiftlog-owned tables
```

All of these are accessed by the same Django project/database connection.

---

# 24. CURRENT-SHIFT EDITING

The `CREATE / EDIT` dashboard actions operate on the current active shift.

Do not make the user manually choose a historical shift for normal current-shift editing.

The backend should determine:

```text
current shift
+
current responsible user
+
editable state
```

and route to the correct log.

Historical signed logs are for viewing, not ordinary editing.

---

# 25. TESTING

Test the database boundary explicitly.

### Database tests

Verify that:

- Shiftlog FK relationships point to `core` models;
- ParameterReading references `core.Area`, `core.System`, `core.Equipment`, and `core.Parameter`;
- SurveillanceExecution references `core.Area`, `core.System`, `core.Equipment`, and `core.Surveillance`;
- migrations create Shiftlog tables only;
- no Shiftlog duplicate master tables are created;
- parameter readings preserve historical values;
- surveillance executions reference the existing `core` definitions.

### PostgreSQL integration

The project's normal test/integration configuration should use PostgreSQL where required to validate the real relational behavior.

Do not make tests depend on the uploaded `db.sqlite3`.

Do not write tests whose setup requires that SQLite file to exist.

### Model tests

Test:

```text
shift timing
night shift crossing midnight
assignments
log relationships
reading history
surveillance execution
handover states
```

### Permission tests

Test:

```text
assigned SE → current shift editable
outgoing SE after handover → current edit denied
incoming SE → current edit allowed
assigned CR → CR log editable
view-only user → edit denied
direct unauthorized URL → denied
```

### Service tests

Test:

```text
create/update log
record reading
execute surveillance
complete handover
transfer responsibility
sign/finalize
```

### Selector tests

Test:

```text
current shift
latest reading
operational status
surveillance counts
recent signed logs
```

### View/dashboard tests

Test:

```text
dashboard renders for SE
dashboard renders for CR
SE controls appear only for appropriate user
CR controls appear only for appropriate user
unauthorized edit is denied
current-shift routing works
```

---

# 26. IMPLEMENTATION ORDER

Follow this order:

```text
1. Inspect existing repository/settings/docs
        ↓
2. Verify PostgreSQL is the active/default database
        ↓
3. Do NOT configure/use db.sqlite3
        ↓
4. Create Shiftlog app
        ↓
5. Define Shiftlog model boundary
        ↓
6. Reference existing core models
        ↓
7. Create Shiftlog migrations
        ↓
8. Implement role + shift assignment + authorization
        ↓
9. Implement forms
        ↓
10. Implement services
        ↓
11. Implement selectors
        ↓
12. Implement dashboard context
        ↓
13. Implement dashboard.html extending home.html
        ↓
14. Implement SE/CR workflows
        ↓
15. Implement Shiftlog-specific CSS
        ↓
16. Add tests
        ↓
17. Run checks/migrations/tests against PostgreSQL (with database content as db.sqlite3)
```

---

# 27. REQUIRED PRE-CODE INSPECTION

Before writing code, inspect:

```text
mysite/settings.py
.env / .env.example
core/models.py
core/migrations/
UserAuthentication/models.py
UserAuthentication/templates/UserAuthentication/home.html
base_generic.html
existing CSS/design tokens
docs/STYLING_GUIDE.md
docs/LAYOUT_LIBRARY.md
existing tests
existing URL configuration
```

Determine exactly how the existing project selects PostgreSQL.

If the repository contains commented SQLite configuration, **do not reactivate it**.

The uploaded `db.sqlite3` is not the target database.

---

# 28. FINAL REPORT FROM CLAUDE

After implementation, report:

```text
Files created
Files TO BE modified
Migrations created
Models introduced
Core models referenced
PostgreSQL/database assumptions
URLs added
Permissions implemented
Shift assignment rules
Handover state transitions
Dashboard context
CSS components created
Tests created
Commands run
Test/check results
Unresolved assumptions
**Files/folders TO BE modified/created in sequence for repo implementation**
```

Explicitly state:

```text
db.sqlite3 was NOT used as the runtime database.
```

If PostgreSQL could not be verified, do not claim successful database integration.

---

# 29. SOURCE-OF-TRUTH SUMMARY

Keep this mental model throughout implementation:

```text
                 EXISTING POSTGRESQL
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
          CORE DATA            SHIFTLOG DATA
             │                     │
             │                     │
      What exists?           What happened?
             │                     │
             │                     │
             │                     ├─ Shift
             │                     ├─ Assignments
             │                     ├─ SE Log
             ├─ Area ─────────────►├─ Area names/references from core app
             ├─ System ───────────►├─ System name, associated area from core app
             ├─ Equipment ────────►├─ Equipment names, associated system and area from core app
             ├─ Parameter ────────►├─ Parameter Reading
             └─ Surveillance ─────►├─ Surveillance Execution
                                   ├─ CR Area Log
                                   └─ Handover
```

**Core owns definitions. Shiftlog owns historical operational events and records. Both live in the existing PostgreSQL database.**

Do not turn the reference SQLite copy into a second source of truth.
