# Building Shiftlog, One File at a Time

This is the code-level companion to `SHIFTLOG_IMPLEMENTATION_GUIDE.md`. Every step names the exact file, gives you the code to put in it, explains *why* it's written that way, tells you what command to run, and what you should see happen.

**Scope of this file:** Milestones 0–5 (scaffold through surveillance execution) are given in full, working code, because they establish every pattern the rest of the app reuses — the `core` boundary, the permission shape, the service/selector split, and the template inheritance chain. Milestones 6–10 (SE log signing, handover, CR log, CSS polish, final report) are given as precise contracts — model fields, service signatures, file lists — rather than full code, because by then you'll be writing them yourself using the same patterns. Ask for any of those in full code once you're there.

Verified against your actual `mysite` project:
- `core.System.area` and `core.Equipment.system` are **many-to-many**, not foreign keys.
- `core.Surveillance.equipment` is **many-to-many**; `core.Surveillance.parameter` is a **required** FK, and `Surveillance.clean()` already keeps its `area`/`system` in sync with the parameter's.
- `core.Parameter.system` is **nullable** (Reactor Power has none).
- `AUTH_USER_MODEL = 'UserAuthentication.CustomUser'`, with a 5-digit `employee_id`.
- `home.html`'s `content` block is empty; its `extra_css`/`extra_js` blocks each load one file — a child template **replaces** these unless it calls `{{ block.super }}`.
- `DB_ENGINE` in `.env` already selects PostgreSQL; the SQLite branch in `settings.py` is live code, not commented out.

---

## Milestone 0 — Baseline

No new files. Run these once, in order, from the `mysite/` project root:

```bash
git init                          # skip if the real repo already has .git
git add -A
git commit -m "chore: baseline before shiftlog"
git switch -c feature/shiftlog
```

Open `mysite/settings.py`, delete the two commented-out credential blocks (the Gmail app password and the hardcoded DB password near the top), rotate both externally, and:

```bash
git add mysite/settings.py
git commit -m "chore: remove committed credentials"
```

Then:

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py loaddata postgres_data.json
python manage.py check
python manage.py test
```

**Expect:** `check` reports no issues; `test` passes (note the count — you'll compare against it after every milestone). If `test` fails here, stop and fix it before writing any Shiftlog code — you need a known-good baseline to know what you broke later.

Also find the real name of the latest `auth` app migration, you'll need it in Milestone 3:

```bash
python manage.py showmigrations auth
```

Note the last line (e.g. `[X] 0012_alter_user_first_name_max_length`) — write it down.

---

## Milestone 1 — Scaffold and the PostgreSQL guard

### Step 1.1 — Create the app

```bash
python manage.py startapp Shiftlog
```

### Step 1.2 — Configure `Shiftlog/apps.py`

```python
# Shiftlog/apps.py
from django.apps import AppConfig


class ShiftlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "Shiftlog"
    label = "shiftlog"
    verbose_name = "Shift Log"

    def ready(self):
        from . import checks  # noqa: F401 — importing registers the check below
```

`ready()` runs once, when Django loads the app registry. Importing `checks` here — rather than at the top of the file — is the standard way to register a system check without creating an import-order problem, since `checks.py` will import `django.db.connection`, which isn't safe to touch before app loading finishes.

### Step 1.3 — Register the app

In `mysite/settings.py`, inside `INSTALLED_APPS`, add the new app **after** `core`:

```python
    'core.apps.CoreConfig',
    'Shiftlog.apps.ShiftlogConfig',
    'CoreManagement.apps.CoremanagementConfig',
```

### Step 1.4 — The PostgreSQL-only guard

```python
# Shiftlog/checks.py
"""
Enforces the project's non-negotiable rule: Shiftlog must never run
against SQLite. `manage.py check` runs this automatically on every
`runserver`, `migrate`, and `test` invocation, so a misconfigured
DB_ENGINE is caught immediately instead of silently writing to the
wrong database.
"""
from django.core.checks import Error, register
from django.db import connection


@register()
def postgresql_only_check(app_configs, **kwargs):
    errors = []
    if connection.vendor != "postgresql":
        errors.append(
            Error(
                "Shiftlog requires PostgreSQL as the active database.",
                hint=(
                    "Set DB_ENGINE=postgresql in your .env file. "
                    "Shiftlog must never run against SQLite, per project policy."
                ),
                obj="Shiftlog",
                id="shiftlog.E001",
            )
        )
    return errors
```

`connection.vendor` is a class attribute of the backend module Django loaded from `DATABASES['default']['ENGINE']` — reading it does **not** open a network connection, so this check is cheap and safe to run on every command, even before a database is reachable.

### Step 1.5 — Clean up the generated scaffold

`startapp` generated a flat `views.py`. Delete it — you're replacing it with a package, because the spec's file layout (`views/dashboard.py`, `views/se_logs.py`, …) needs `views/` to be a package, and Python can't have both a `views.py` module and a `views/` package with the same name.

```bash
rm Shiftlog/views.py
mkdir -p Shiftlog/views Shiftlog/templates/Shiftlog Shiftlog/static/Shiftlog/css/layout Shiftlog/static/Shiftlog/css/components Shiftlog/tests
touch Shiftlog/views/__init__.py Shiftlog/tests/__init__.py
```

Also delete the generated `Shiftlog/tests.py` (you just created a `tests/` package instead):

```bash
rm -f Shiftlog/tests.py
```

### Step 1.6 — Test the guard

```python
# Shiftlog/tests/test_checks.py
from unittest import mock

from django.test import SimpleTestCase

from Shiftlog.checks import postgresql_only_check


class PostgresOnlyCheckTests(SimpleTestCase):
    def test_passes_on_postgresql(self):
        with mock.patch("Shiftlog.checks.connection") as mock_conn:
            mock_conn.vendor = "postgresql"
            self.assertEqual(postgresql_only_check(app_configs=None), [])

    def test_fails_on_sqlite(self):
        with mock.patch("Shiftlog.checks.connection") as mock_conn:
            mock_conn.vendor = "sqlite"
            errors = postgresql_only_check(app_configs=None)
            self.assertEqual(len(errors), 1)
            self.assertEqual(errors[0].id, "shiftlog.E001")
```

`SimpleTestCase` (not `TestCase`) is deliberate — this test never touches the database, so it doesn't need Django's per-test transaction wrapping, and it runs faster.

### Step 1.7 — Run and verify

```bash
python manage.py check
python manage.py test Shiftlog
```

**Expect:** `check` still reports no issues (you're on Postgres). Both new tests pass. As a sanity check that the guard actually works, temporarily set `DB_ENGINE=sqlite` in `.env`, run `python manage.py check` again, confirm you see `shiftlog.E001`, then set `.env` back to `postgresql`.

### Commit

```bash
git add Shiftlog mysite/settings.py
git commit -m "feat(shiftlog): scaffold app and add postgres-only system check"
```

---

## Milestone 2 — Skeleton dashboard, then flip the landing page

### Step 2.1 — The dashboard view

```python
# Shiftlog/views/dashboard.py
from django.shortcuts import render
from django.views.decorators.cache import never_cache

from UserAuthentication.decorators import login_required


@login_required
@never_cache
def dashboard(request):
    context = {}
    return render(request, "Shiftlog/dashboard.html", context)
```

Reusing `UserAuthentication.decorators.login_required` (not `django.contrib.auth.decorators.login_required` directly) matters: it's the wrapper that adds the project's "Please login" flash message while still giving you Django's `next=` redirect behaviour — see the docstring in that file. `never_cache` matches `home`'s own view, so the back button never shows a cached page after logout.

### Step 2.2 — URLs

```python
# Shiftlog/urls.py
from django.urls import path

from Shiftlog.views.dashboard import dashboard

app_name = "shiftlog"

urlpatterns = [
    path("", dashboard, name="dashboard"),
]
```

`app_name = "shiftlog"` namespaces every URL name in this file, so `{% url 'shiftlog:dashboard' %}` is unambiguous even though `core/urls.py` also happens to define a route named `dashboard`.

In `mysite/urls.py`, add the include:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('shiftlog/', include('Shiftlog.urls')),
    path('', include('UserAuthentication.urls'), name='UserAuthentication'),
]
```

### Step 2.3 — The dashboard template

```html
{# Shiftlog/templates/Shiftlog/dashboard.html #}
{% extends "UserAuthentication/home.html" %}
{% load static %}

{% block title %}DRISHTI — Shift Dashboard{% endblock %}
{% block page_title %}Shift Dashboard{% endblock %}

{% block extra_css %}
  {{ block.super }}
  <link rel="stylesheet" href="{% static 'Shiftlog/css/layout/dashboard.css' %}">
{% endblock %}

{% block content %}
<div class="shiftlog-dashboard">
  <p>Shiftlog dashboard — under construction.</p>
</div>
{% endblock %}

{% block extra_js %}
  {{ block.super }}
{% endblock %}
```

Two things here are load-bearing, not stylistic:

1. **`{{ block.super }}` in `extra_css`.** `home.html` uses this same block to load `home.css`. Django block overriding *replaces* the parent's content by default — without `{{ block.super }}`, your dashboard would silently lose `home.css` (and with it, most of the layout) the moment you extend `home.html`.
2. **`{% load static %}` at the top of *this* file.** Template tag loading isn't inherited across `{% extends %}` — `home.html` loading `static` doesn't give this file access to `{% static %}`.

### Step 2.4 — A minimal stylesheet

```css
/* Shiftlog/static/Shiftlog/css/layout/dashboard.css */
.shiftlog-dashboard {
  padding: var(--sp-6);
}
```

This exists mainly to prove the static file pipeline works end to end before you build anything real on top of it.

### Step 2.5 — Test the skeleton

```python
# Shiftlog/tests/test_dashboard.py
from django.contrib.auth import get_user_model
from django.test import TestCase
from django.urls import reverse


class DashboardViewTests(TestCase):
    def setUp(self):
        self.user = get_user_model().objects.create_user(
            username="se_test", password="pass12345", employee_id="10001"
        )

    def test_anonymous_redirects_to_login(self):
        response = self.client.get(reverse("shiftlog:dashboard"))
        self.assertRedirects(
            response,
            f"{reverse('login')}?next={reverse('shiftlog:dashboard')}",
        )

    def test_authenticated_user_gets_200(self):
        self.client.force_login(self.user)
        response = self.client.get(reverse("shiftlog:dashboard"))
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, "Shiftlog/dashboard.html")

    def test_response_is_not_cached(self):
        self.client.force_login(self.user)
        response = self.client.get(reverse("shiftlog:dashboard"))
        self.assertIn("no-cache", response.headers.get("Cache-Control", ""))
```

### Step 2.6 — Run, check by hand, commit A

```bash
python manage.py test Shiftlog
python manage.py runserver
```

Visit `http://localhost:8000/shiftlog/`, log in, and confirm the header, clock, theme toggle and feedback widget are all present — that tells you `home.css`/`home.js` survived the inheritance.

```bash
git add Shiftlog mysite/urls.py
git commit -m "feat(shiftlog): skeleton dashboard extending home.html"
```

### Step 2.7 — Flip the landing page

Edit `UserAuthentication/views/home_views.py`. Replace its body with a delegation to the Shiftlog dashboard, **keeping the file's own decorators**:

```python
# UserAuthentication/views/home_views.py
from django.views.decorators.cache import never_cache

from UserAuthentication.decorators import login_required


@login_required
@never_cache
def home(request):
    from Shiftlog.views.dashboard import dashboard  # local import avoids a
    # circular import: UserAuthentication.urls is loaded very early (it's
    # the root include in mysite/urls.py), and importing Shiftlog.views at
    # module level here would run before Shiftlog's own URLs are ready.
    return dashboard(request)
```

You're not deleting `home.html`, `home.css`, `home.js` or the `home` URL name — only the Python view body changes. Every existing reference to `{% url 'home' %}`, `reverse('home')`, and `LOGIN_REDIRECT_URL = 'home'` keeps working unmodified.

### Step 2.8 — Add a regression test and rerun the whole suite

```python
# add to Shiftlog/tests/test_dashboard.py
from django.test import TestCase
from django.urls import reverse
from django.contrib.auth import get_user_model


class HomeDelegatesToShiftlogTests(TestCase):
    def test_home_renders_shiftlog_dashboard(self):
        user = get_user_model().objects.create_user(
            username="se_home_test", password="pass12345", employee_id="10002"
        )
        self.client.force_login(user)
        response = self.client.get(reverse("home"))
        self.assertTemplateUsed(response, "Shiftlog/dashboard.html")
```

```bash
python manage.py test
```

**Expect:** the full suite passes, including every pre-existing `UserAuthentication` test (login, registration, logout) — those never touch the view body you changed, only the URL name, which is untouched. Log in through the browser; you should land on `/home/` and see the same skeleton dashboard.

### Commit B

```bash
git add UserAuthentication/views/home_views.py Shiftlog/tests/test_dashboard.py
git commit -m "feat(shiftlog): render shiftlog dashboard as the landing page"
```

---

## Milestone 3 — Shifts, assignments, roles, on-duty header

### Step 3.1 — Models

```python
# Shiftlog/models.py
from django.conf import settings
from django.db import models


class TimeStamped(models.Model):
    """
    Local to Shiftlog on purpose — Shiftlog doesn't import core's
    TimeStampedModel, keeping the two apps' base classes independent
    even though they look alike.
    """
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Shift(TimeStamped):
    class ShiftType(models.TextChoices):
        DAY = "DAY", "Day (07:00–15:00)"
        EVENING = "EVENING", "Evening (15:00–23:00)"
        NIGHT = "NIGHT", "Night (23:00–07:00)"

    class Status(models.TextChoices):
        SCHEDULED = "SCHEDULED", "Scheduled"
        ACTIVE = "ACTIVE", "Active"
        CLOSED = "CLOSED", "Closed"

    date = models.DateField(
        help_text=(
            "The shift's start date. A NIGHT shift starting 23:00 on this "
            "date ends 07:00 the following calendar day."
        )
    )
    shift_type = models.CharField(max_length=10, choices=ShiftType.choices)
    scheduled_start = models.DateTimeField()
    scheduled_end = models.DateTimeField()
    actual_start = models.DateTimeField(null=True, blank=True)
    actual_end = models.DateTimeField(null=True, blank=True)
    status = models.CharField(
        max_length=10, choices=Status.choices, default=Status.SCHEDULED
    )

    class Meta:
        ordering = ["-date", "shift_type"]
        constraints = [
            models.UniqueConstraint(
                fields=["date", "shift_type"], name="unique_shift_per_date_type"
            ),
        ]

    def __str__(self):
        return f"{self.date} {self.get_shift_type_display()}"


class ShiftAssignment(TimeStamped):
    class Role(models.TextChoices):
        SE = "SE", "Shift Engineer"
        CR = "CR", "Control Room Operator"

    shift = models.ForeignKey(
        Shift, on_delete=models.CASCADE, related_name="assignments"
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.PROTECT,
        related_name="shift_assignments",
    )
    role = models.CharField(max_length=2, choices=Role.choices)
    is_current = models.BooleanField(default=True)
    assigned_from = models.DateTimeField(auto_now_add=True)
    assigned_until = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ["-assigned_from"]
        constraints = [
            models.UniqueConstraint(
                fields=["shift", "role"],
                condition=models.Q(is_current=True, role="SE"),
                name="unique_current_se_per_shift",
            ),
        ]

    def __str__(self):
        return f"{self.user} as {self.get_role_display()} on {self.shift}"
```

Why `shift` uses `CASCADE` here but every `core` FK you'll write from Milestone 4 onward uses `PROTECT`: a `ShiftAssignment` has no meaning without its `Shift` — deleting the shift should delete its assignments. A `ParameterReading`, by contrast, is a historical fact that must outlive accidental deletion of the plant record it points to — that's what `PROTECT` is for. Same reasoning applies to `Shift` itself as a FK target from readings and executions later: those use `PROTECT` too, because a shift row shouldn't be deletable once it has recorded history.

The `UniqueConstraint` with a `condition` is a **partial unique index** — Postgres enforces "at most one current SE per shift" at the database level, not just in application code. Multiple `is_current=True` CR rows are allowed because the condition only matches `role="SE"`.

### Step 3.2 — Role name constants

```python
# Shiftlog/roles.py
SHIFT_ENGINEER_GROUP = "SHIFT_ENGINEER"
CONTROL_ROOM_OPERATOR_GROUP = "CONTROL_ROOM_OPERATOR"
```

A separate module (rather than string literals scattered through `permissions.py`, the admin, and a data migration) means the group name is a single source of truth — a typo in one place would silently create a third, empty group.

### Step 3.3 — Migrations

```bash
python manage.py makemigrations Shiftlog
```

**Before applying it**, open `Shiftlog/migrations/0001_initial.py` and check:
- it only creates `shiftlog_shift` and `shiftlog_shiftassignment`;
- its `dependencies` list includes `('UserAuthentication', '__first__')` or the specific migration Django picked automatically (it resolves `settings.AUTH_USER_MODEL` for you — you don't write this by hand, just confirm it's there);
- no `core` table appears anywhere in it.

```bash
python manage.py sqlmigrate Shiftlog 0001
```

Read the printed SQL — you should see exactly two `CREATE TABLE` statements plus the constraint. Then:

```bash
python manage.py migrate Shiftlog
```

### Step 3.4 — Data migration for the groups

```bash
python manage.py makemigrations Shiftlog --empty --name create_role_groups
```

Edit the generated file:

```python
# Shiftlog/migrations/0002_create_role_groups.py
from django.db import migrations

from Shiftlog.roles import SHIFT_ENGINEER_GROUP, CONTROL_ROOM_OPERATOR_GROUP

GROUP_NAMES = [SHIFT_ENGINEER_GROUP, CONTROL_ROOM_OPERATOR_GROUP]


def create_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    for name in GROUP_NAMES:
        Group.objects.get_or_create(name=name)


def remove_groups(apps, schema_editor):
    Group = apps.get_model("auth", "Group")
    Group.objects.filter(name__in=GROUP_NAMES).delete()


class Migration(migrations.Migration):
    dependencies = [
        ("shiftlog", "0001_initial"),
        # Replace with the migration name you noted from
        # `python manage.py showmigrations auth` in Milestone 0.
        ("auth", "0012_alter_user_first_name_max_length"),
    ]

    operations = [
        migrations.RunPython(create_groups, remove_groups),
    ]
```

`apps.get_model("auth", "Group")` (the *historical* model, from the migration's frozen app registry) rather than `from django.contrib.auth.models import Group` is the standard pattern for data migrations — it keeps this migration correct even if a future Django version changes the real `Group` model, because migrations replay against the schema as it existed at that point in history, not against today's code.

```bash
python manage.py migrate Shiftlog
```

**Expect:** `python manage.py shell -c "from django.contrib.auth.models import Group; print(list(Group.objects.values_list('name', flat=True)))"` prints both group names.

### Step 3.5 — Selectors

```python
# Shiftlog/selectors.py
"""
Read-only queries. Nothing in this module writes to the database —
that discipline is what lets views and templates call these freely
without worrying about side effects.
"""
from .models import Shift, ShiftAssignment


def get_current_shift():
    return Shift.objects.filter(status=Shift.Status.ACTIVE).order_by("-date").first()


def get_current_responsible_se(shift):
    if shift is None:
        return None
    assignment = (
        ShiftAssignment.objects.filter(
            shift=shift, role=ShiftAssignment.Role.SE, is_current=True
        )
        .select_related("user")
        .first()
    )
    return assignment.user if assignment else None


def get_current_cr_operators(shift):
    if shift is None:
        return []
    return [
        a.user
        for a in ShiftAssignment.objects.filter(
            shift=shift, role=ShiftAssignment.Role.CR, is_current=True
        ).select_related("user")
    ]
```

### Step 3.6 — Permissions (first cut)

```python
# Shiftlog/permissions.py
"""
Pure functions: (user, shift, ...) -> bool. These are the authorization
boundary — every mutating view must call one of these BEFORE touching
any data. Template {% if %} checks that reuse the same functions are
for hiding buttons, not for security.
"""
from .roles import SHIFT_ENGINEER_GROUP, CONTROL_ROOM_OPERATOR_GROUP
from .selectors import get_current_responsible_se


def is_shift_engineer(user):
    return user.groups.filter(name=SHIFT_ENGINEER_GROUP).exists()


def is_control_room_operator(user):
    return user.groups.filter(name=CONTROL_ROOM_OPERATOR_GROUP).exists()


def can_edit_se_log(user, shift):
    if shift is None or shift.status != Shift_Status_ACTIVE(shift):
        return False
    if not is_shift_engineer(user):
        return False
    return get_current_responsible_se(shift) == user


def Shift_Status_ACTIVE(shift):
    # Tiny helper so the comparison above reads naturally; equivalent to
    # `shift.status != shift.Status.ACTIVE` but avoids importing Shift here.
    return shift.status == "ACTIVE"
```

*(If the helper function at the bottom looks awkward to you, that's a fair reaction — it exists only to avoid a second import line. Feel free to simplify it to `shift.status != shift.Status.ACTIVE` directly and delete the helper; both are correct.)*

### Step 3.7 — Wire the on-duty header into the dashboard

```python
# Shiftlog/views/dashboard.py — replace the whole file
from django.shortcuts import render
from django.views.decorators.cache import never_cache

from UserAuthentication.decorators import login_required

from Shiftlog.selectors import get_current_shift, get_current_responsible_se, get_current_cr_operators


@login_required
@never_cache
def dashboard(request):
    shift = get_current_shift()
    context = {
        "shift": shift,
        "responsible_se": get_current_responsible_se(shift),
        "cr_operators": get_current_cr_operators(shift),
    }
    return render(request, "Shiftlog/dashboard.html", context)
```

```html
{# Shiftlog/templates/Shiftlog/partials/on_duty_header.html #}
<section class="on-duty-header">
  {% if shift %}
    <span class="on-duty-header__shift">{{ shift.get_shift_type_display }} — {{ shift.date }}</span>
    <span class="on-duty-header__se">
      SE: {% if responsible_se %}{{ responsible_se.get_full_name|default:responsible_se.username }}{% else %}<em>unassigned</em>{% endif %}
    </span>
    {% if cr_operators %}
      <span class="on-duty-header__cr">CR: {% for cr in cr_operators %}{{ cr.get_full_name|default:cr.username }}{% if not forloop.last %}, {% endif %}{% endfor %}</span>
    {% endif %}
  {% else %}
    <span class="on-duty-header__none">No active shift.</span>
  {% endif %}
</section>
```

```html
{# Shiftlog/templates/Shiftlog/dashboard.html — update the content block #}
{% block content %}
<div class="shiftlog-dashboard">
  {% include "Shiftlog/partials/on_duty_header.html" %}
</div>
{% endblock %}
```

### Step 3.8 — Admin registration

```python
# Shiftlog/admin.py
from django.contrib import admin

from .models import Shift, ShiftAssignment


@admin.register(Shift)
class ShiftAdmin(admin.ModelAdmin):
    list_display = ("pk", "date", "shift_type", "status", "scheduled_start", "scheduled_end")
    list_filter = ("shift_type", "status")
    ordering = ("-date", "shift_type")


@admin.register(ShiftAssignment)
class ShiftAssignmentAdmin(admin.ModelAdmin):
    list_display = ("pk", "shift", "user", "role", "is_current", "assigned_from")
    list_filter = ("role", "is_current")
    ordering = ("-assigned_from",)
```

### Step 3.9 — Tests

```python
# Shiftlog/tests/test_models.py
from datetime import datetime, timedelta

from django.contrib.auth import get_user_model
from django.db import IntegrityError, transaction
from django.test import TestCase
from django.utils import timezone

from Shiftlog.models import Shift, ShiftAssignment


class ShiftTests(TestCase):
    def test_night_shift_crosses_midnight(self):
        start = timezone.make_aware(datetime(2026, 9, 28, 23, 0))
        end = start + timedelta(hours=8)  # 07:00 the next calendar day
        shift = Shift.objects.create(
            date=start.date(),  # keyed to the START date, per the spec
            shift_type=Shift.ShiftType.NIGHT,
            scheduled_start=start,
            scheduled_end=end,
        )
        self.assertEqual(shift.date.day, 28)
        self.assertEqual(shift.scheduled_end.day, 29)

    def test_one_shift_per_date_and_type(self):
        Shift.objects.create(
            date="2026-09-28",
            shift_type=Shift.ShiftType.DAY,
            scheduled_start=timezone.now(),
            scheduled_end=timezone.now() + timedelta(hours=8),
        )
        with self.assertRaises(IntegrityError):
            with transaction.atomic():
                Shift.objects.create(
                    date="2026-09-28",
                    shift_type=Shift.ShiftType.DAY,
                    scheduled_start=timezone.now(),
                    scheduled_end=timezone.now() + timedelta(hours=8),
                )


class ShiftAssignmentTests(TestCase):
    def setUp(self):
        self.shift = Shift.objects.create(
            date="2026-09-28",
            shift_type=Shift.ShiftType.DAY,
            scheduled_start=timezone.now(),
            scheduled_end=timezone.now() + timedelta(hours=8),
            status=Shift.Status.ACTIVE,
        )
        self.se1 = get_user_model().objects.create_user(
            username="se1", password="pass12345", employee_id="10003"
        )
        self.se2 = get_user_model().objects.create_user(
            username="se2", password="pass12345", employee_id="10004"
        )

    def test_only_one_current_se_per_shift(self):
        ShiftAssignment.objects.create(
            shift=self.shift, user=self.se1, role=ShiftAssignment.Role.SE, is_current=True
        )
        with self.assertRaises(IntegrityError):
            with transaction.atomic():
                ShiftAssignment.objects.create(
                    shift=self.shift, user=self.se2, role=ShiftAssignment.Role.SE, is_current=True
                )
```

```python
# Shiftlog/tests/test_selectors.py
from datetime import timedelta

from django.contrib.auth import get_user_model
from django.test import TestCase
from django.utils import timezone

from Shiftlog.models import Shift, ShiftAssignment
from Shiftlog.selectors import get_current_shift, get_current_responsible_se


class SelectorTests(TestCase):
    def test_no_active_shift_returns_none(self):
        self.assertIsNone(get_current_shift())
        self.assertIsNone(get_current_responsible_se(None))

    def test_returns_the_active_shift_and_its_se(self):
        shift = Shift.objects.create(
            date="2026-09-28",
            shift_type=Shift.ShiftType.DAY,
            scheduled_start=timezone.now(),
            scheduled_end=timezone.now() + timedelta(hours=8),
            status=Shift.Status.ACTIVE,
        )
        se = get_user_model().objects.create_user(
            username="se_sel", password="pass12345", employee_id="10005"
        )
        ShiftAssignment.objects.create(
            shift=shift, user=se, role=ShiftAssignment.Role.SE, is_current=True
        )
        self.assertEqual(get_current_shift(), shift)
        self.assertEqual(get_current_responsible_se(shift), se)
```

### Step 3.10 — Run, verify, commit

```bash
python manage.py makemigrations --check --dry-run
python manage.py test Shiftlog
```

**Expect:** no missing migrations reported; all new tests pass. In the admin, create the two test users (5-digit `employee_id`, not superusers), put one in each group, create a `Shift` with `status=ACTIVE`, and assign the SE. Reload `/shiftlog/` — the on-duty header should show the shift and the SE's name.

```bash
git add Shiftlog
git commit -m "feat(shiftlog): shift and assignment models with role groups"
git commit -am "feat(shiftlog): on-duty header on dashboard"   # if you split the header out separately
```

(If you did steps 3.7–3.8 in the same working session as 3.1–3.6, one combined commit is fine too — the two-commit split above is only useful if you want the header as a reviewable, separate diff.)

---

## Milestone 4 — Parameter readings and operational status

### Step 4.1 — Model

Append to `Shiftlog/models.py`:

```python
from django.utils import timezone as dj_timezone


class ParameterReading(TimeStamped):
    class Source(models.TextChoices):
        SE_MANUAL = "SE_MANUAL", "SE Manual Entry"
        # CR_MANUAL, CALCULATED and SYSTEM are deliberately not added yet —
        # see Decision 4 in the implementation guide. Add them only when a
        # concrete workflow needs them.

    shift = models.ForeignKey(Shift, on_delete=models.PROTECT, related_name="parameter_readings")
    area = models.ForeignKey(
        "core.Area", on_delete=models.PROTECT, related_name="shiftlog_parameter_readings"
    )
    system = models.ForeignKey(
        "core.System",
        on_delete=models.PROTECT,
        related_name="shiftlog_parameter_readings",
        null=True,
        blank=True,
    )
    equipment = models.ForeignKey(
        "core.Equipment", on_delete=models.PROTECT, related_name="shiftlog_parameter_readings"
    )
    parameter = models.ForeignKey(
        "core.Parameter", on_delete=models.PROTECT, related_name="shift_readings"
    )
    value = models.FloatField()
    recorded_at = models.DateTimeField(default=dj_timezone.now)
    source = models.CharField(max_length=20, choices=Source.choices, default=Source.SE_MANUAL)
    recorded_by = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="parameter_readings"
    )

    class Meta:
        ordering = ["-recorded_at"]
        indexes = [models.Index(fields=["parameter", "recorded_at"])]

    def __str__(self):
        return f"{self.parameter.name} = {self.value} @ {self.recorded_at:%Y-%m-%d %H:%M}"
```

`system` is nullable because `core.Parameter.system` is nullable in your real data (Reactor Power has none) — a reading copies that nullability, it doesn't fight it. `on_delete=PROTECT` on all four `core` FKs is the spec's central rule made concrete: Postgres will refuse to delete an `Area`, `System`, `Equipment` or `Parameter` that any reading still points to, so history can never be silently orphaned.

### Step 4.2 — Migration

```bash
python manage.py makemigrations Shiftlog
```

Open the generated file and check the two `core` FKs are declared with `to='core.area'` etc. (lowercase — Django lowercases app/model labels internally; this is normal) and that the migration's `dependencies` list includes `core`'s latest migration.

```bash
python manage.py sqlmigrate Shiftlog 0003
python manage.py migrate Shiftlog
```

### Step 4.3 — Named parameter lookup (no primary keys, ever)

```python
# Shiftlog/dashboard_parameters.py
"""
Named lookup for the five core.Parameter records the dashboard shows,
resolved by stable attributes — never by primary key, per the spec's
explicit ban on hard-coded IDs (SQLite PKs won't match Postgres PKs).

If your real core data has more than one active Parameter with the same
name, add an 'equipment__name' or 'area__name' key to the relevant dict
below. get_dashboard_parameter() will raise a clear LookupError telling
you exactly which key needs narrowing, instead of guessing.
"""

DASHBOARD_PARAMETERS = {
    "reactor_power": {"name": "Reactor Power"},
    "activity_readings": {"name": "Activity Readings"},
    "trips_and_shutdowns": {"name": "Trips & Shutdowns"},
    "af_cf": {"name": "AF CF"},
    "exhaust_air_activity": {"name": "Exhaust Air Activity"},
}
```

```python
# Shiftlog/selectors.py — append
from django.core.exceptions import MultipleObjectsReturned, ObjectDoesNotExist

from core.models import Parameter

from .dashboard_parameters import DASHBOARD_PARAMETERS
from .models import ParameterReading


def get_dashboard_parameter(key):
    lookup = DASHBOARD_PARAMETERS[key]
    try:
        return Parameter.objects.get(is_active=True, **lookup)
    except ObjectDoesNotExist:
        raise LookupError(
            f"No active core.Parameter matches {lookup!r} for dashboard key '{key}'. "
            "Check the name in the admin, or adjust dashboard_parameters.py."
        )
    except MultipleObjectsReturned:
        raise LookupError(
            f"Multiple active core.Parameter rows match {lookup!r} for dashboard key "
            f"'{key}'. Add equipment__name or area__name to disambiguate."
        )


def get_latest_parameter_reading(parameter):
    return ParameterReading.objects.filter(parameter=parameter).order_by("-recorded_at").first()


def get_operational_status():
    status = {}
    for key in DASHBOARD_PARAMETERS:
        try:
            parameter = get_dashboard_parameter(key)
        except LookupError as exc:
            status[key] = {"error": str(exc)}
            continue
        status[key] = {
            "parameter": parameter,
            "reading": get_latest_parameter_reading(parameter),
        }
    return status
```

This selector is deliberately loud on failure. A silent `.first()` that returns `None` on a bad lookup would make a misconfigured parameter name look identical to "no reading yet" on the dashboard — the `LookupError` path keeps those two situations visibly different.

### Step 4.4 — Service

```python
# Shiftlog/services.py
"""
Mutating workflows. Every function here is @transaction.atomic and
re-checks permission itself — never trust that a view already checked,
since services may eventually be called from the admin or a shell too.
"""
from django.db import transaction

from .models import ParameterReading
from .permissions import can_edit_se_log
from .selectors import get_current_shift


@transaction.atomic
def record_parameter_reading(*, user, parameter, value, area=None, system=None, equipment=None):
    shift = get_current_shift()
    if shift is None:
        raise ValueError("No active shift — cannot record a reading.")
    if not can_edit_se_log(user, shift):
        raise PermissionError("Only the current responsible SE can record readings.")

    area = area or parameter.area
    equipment = equipment or parameter.equipment
    system = system if system is not None else parameter.system

    if equipment.area_id != area.id:
        raise ValueError("Equipment does not belong to the given area.")

    return ParameterReading.objects.create(
        shift=shift,
        area=area,
        system=system,
        equipment=equipment,
        parameter=parameter,
        value=value,
        source=ParameterReading.Source.SE_MANUAL,
        recorded_by=user,
    )
```

Defaulting `area`/`equipment`/`system` from the parameter itself (rather than requiring the caller to always pass them) keeps the common case — "record a reading for this exact parameter" — a one-line call, while still letting a future multi-location parameter override them explicitly.

### Step 4.5 — Template partial

```html
{# Shiftlog/templates/Shiftlog/partials/operational_status.html #}
<section class="metric-grid">
  {% for key, entry in operational_status.items %}
    <div class="metric-card{% if entry.error %} metric-card--error{% endif %}">
      {% if entry.error %}
        <div class="metric-card__label">{{ key }}</div>
        <div class="metric-card__value metric-card__value--error">Configuration issue</div>
      {% else %}
        <div class="metric-card__label">{{ entry.parameter.name }}</div>
        {% if entry.reading %}
          <div class="metric-card__value">{{ entry.reading.value }} {{ entry.parameter.unit }}</div>
          <div class="metric-card__meta">as of {{ entry.reading.recorded_at|date:"H:i" }}</div>
        {% else %}
          <div class="metric-card__value metric-card__value--muted">No reading yet</div>
        {% endif %}
        {% if key == "trips_and_shutdowns" %}
          <div class="metric-card__note">core stores this as one numeric value — see README.</div>
        {% endif %}
      {% endif %}
    </div>
  {% endfor %}
</section>
```

One `.metric-card` component drives all five tiles from data, per Decision 4/5 and the CSS-ownership section of the original spec — there's no `reactor-power-card.css` or `trips-card.css`.

```css
/* Shiftlog/static/Shiftlog/css/components/metric-card.css */
.metric-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: var(--sp-4);
  margin-top: var(--sp-4);
}
.metric-card {
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-lg);
  padding: var(--sp-4);
}
.metric-card--error {
  border-color: var(--color-danger);
  background: var(--color-danger-bg);
}
.metric-card__label {
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
}
.metric-card__value {
  font-size: var(--text-2xl);
  font-weight: var(--weight-semibold);
  color: var(--color-text-primary);
}
.metric-card__value--muted {
  color: var(--color-text-muted);
  font-size: var(--text-lg);
}
.metric-card__meta {
  font-size: var(--text-xs);
  color: var(--color-text-muted);
}
.metric-card__note {
  font-size: var(--text-xs);
  color: var(--color-warning);
  margin-top: var(--sp-2);
}
```

Wire it into the dashboard view and template:

```python
# Shiftlog/views/dashboard.py — add to context
from Shiftlog.selectors import get_operational_status
# ...
    context = {
        "shift": shift,
        "responsible_se": get_current_responsible_se(shift),
        "cr_operators": get_current_cr_operators(shift),
        "operational_status": get_operational_status(),
    }
```

```html
{# dashboard.html content block #}
  {% include "Shiftlog/partials/operational_status.html" %}
```

```html
{# extra_css block — add the new component #}
  <link rel="stylesheet" href="{% static 'Shiftlog/css/components/metric-card.css' %}">
```

### Step 4.6 — Tests

```python
# Shiftlog/tests/test_selectors.py — append
from Shiftlog.selectors import get_dashboard_parameter, get_operational_status


class DashboardParameterTests(TestCase):
    def test_unknown_parameter_name_raises_loudly(self):
        # Assumes your dev DB has no Parameter named this; adjust if it does.
        from Shiftlog.dashboard_parameters import DASHBOARD_PARAMETERS
        DASHBOARD_PARAMETERS["_test_missing"] = {"name": "Does Not Exist"}
        with self.assertRaises(LookupError):
            get_dashboard_parameter("_test_missing")
        del DASHBOARD_PARAMETERS["_test_missing"]
```

```python
# Shiftlog/tests/test_services.py
from datetime import timedelta

from django.contrib.auth import get_user_model
from django.contrib.auth.models import Group
from django.test import TestCase
from django.utils import timezone

from core.models import Area, Equipment, Parameter
from Shiftlog.models import Shift, ShiftAssignment
from Shiftlog.roles import SHIFT_ENGINEER_GROUP
from Shiftlog.services import record_parameter_reading


class RecordParameterReadingTests(TestCase):
    def setUp(self):
        self.area = Area.objects.create(name="Reactor Building", abbreviation="RB")
        self.equipment = Equipment.objects.create(
            name="Reactor", area=self.area, abbreviation="RX", equip_type="OTHER"
        )
        self.parameter = Parameter.objects.create(
            name="Reactor Power", param_type="CALCULATED",
            area=self.area, equipment=self.equipment,
        )
        self.shift = Shift.objects.create(
            date="2026-09-28", shift_type=Shift.ShiftType.DAY,
            scheduled_start=timezone.now(),
            scheduled_end=timezone.now() + timedelta(hours=8),
            status=Shift.Status.ACTIVE,
        )
        se_group = Group.objects.create(name=SHIFT_ENGINEER_GROUP)
        self.se = get_user_model().objects.create_user(
            username="se_svc", password="pass12345", employee_id="10006"
        )
        self.se.groups.add(se_group)
        ShiftAssignment.objects.create(
            shift=self.shift, user=self.se, role=ShiftAssignment.Role.SE, is_current=True
        )

    def test_se_can_record_a_reading(self):
        reading = record_parameter_reading(user=self.se, parameter=self.parameter, value=42.0)
        self.assertEqual(reading.value, 42.0)
        self.assertEqual(reading.recorded_by, self.se)

    def test_non_se_is_refused(self):
        other = get_user_model().objects.create_user(
            username="not_se", password="pass12345", employee_id="10007"
        )
        with self.assertRaises(PermissionError):
            record_parameter_reading(user=other, parameter=self.parameter, value=1.0)
```

### Step 4.7 — Run, verify, commit

```bash
python manage.py test Shiftlog
```

**Expect:** all pass. Manually, from `python manage.py shell`, call `record_parameter_reading` a few times for different parameters logged in as your test SE, then reload the dashboard and confirm the tiles populate and the "No reading yet" / error states look right. Check dark mode.

```bash
git add Shiftlog
git commit -m "feat(shiftlog): parameter reading model and service"
git commit -am "feat(shiftlog): operational status tiles"   # or fold into one commit
```

---

## Milestone 5 — Surveillance execution and status

### Step 5.1 — Setting

```python
# mysite/settings.py — add near the bottom
# Shiftlog: a surveillance becomes "Long Pending" once it has missed this
# many full frequency cycles past its core.Surveillance.next_date.
SHIFTLOG_LONG_PENDING_LAPSES = 10
```

### Step 5.2 — Schedule helper

```python
# Shiftlog/schedule.py
"""
Pure date math, deliberately separate from services.py so it can be
unit-tested without touching the database at all, and so the "as_of"
date is always an explicit argument — never datetime.today() buried
inside a function — which is what makes the tests below deterministic.
"""
from dateutil.relativedelta import relativedelta

from core.models import Surveillance

FREQUENCY_INTERVALS = {
    Surveillance.Frequency.DAILY: relativedelta(days=1),
    Surveillance.Frequency.WEEKLY: relativedelta(weeks=1),
    Surveillance.Frequency.MONTHLY: relativedelta(months=1),
    Surveillance.Frequency.QUARTERLY: relativedelta(months=3),
    Surveillance.Frequency.ANNUAL: relativedelta(years=1),
}


def advance_next_date(next_date, frequency, as_of):
    """New next_date after a due execution recorded on as_of."""
    if frequency == Surveillance.Frequency.SHIFT:
        # SHIFT frequency isn't date-driven (see Decision 1) — leave it as
        # the caller's problem to update, or extend this once needed.
        return next_date
    interval = FREQUENCY_INTERVALS[frequency]
    base = next_date or as_of
    return base + interval


def lapses_overdue(next_date, frequency, as_of):
    """How many full frequency cycles have been missed past next_date."""
    if frequency == Surveillance.Frequency.SHIFT or next_date is None or next_date > as_of:
        return 0
    interval = FREQUENCY_INTERVALS[frequency]
    count = 0
    probe = next_date
    while probe + interval <= as_of:
        probe += interval
        count += 1
    return count
```

### Step 5.3 — Model

```python
# Shiftlog/models.py — append
class SurveillanceExecution(TimeStamped):
    class Result(models.TextChoices):
        SATISFACTORY = "SATISFACTORY", "Satisfactory"
        UNSATISFACTORY = "UNSATISFACTORY", "Unsatisfactory"

    shift = models.ForeignKey(Shift, on_delete=models.PROTECT, related_name="surveillance_executions")
    area = models.ForeignKey(
        "core.Area", on_delete=models.PROTECT, related_name="shiftlog_surveillance_executions"
    )
    system = models.ForeignKey(
        "core.System",
        on_delete=models.PROTECT,
        related_name="shiftlog_surveillance_executions",
        null=True,
        blank=True,
    )
    equipment = models.ForeignKey(
        "core.Equipment", on_delete=models.PROTECT, related_name="shiftlog_surveillance_executions"
    )
    parameter = models.ForeignKey(
        "core.Parameter", on_delete=models.PROTECT, related_name="surveillance_executions"
    )
    surveillance = models.ForeignKey(
        "core.Surveillance", on_delete=models.PROTECT, related_name="shift_executions"
    )
    due_date = models.DateField()
    executed_at = models.DateTimeField(default=dj_timezone.now)
    executor = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="surveillance_executions"
    )
    result = models.CharField(max_length=20, choices=Result.choices)
    remarks = models.TextField(blank=True)

    class Meta:
        ordering = ["-executed_at"]
        constraints = [
            models.UniqueConstraint(
                fields=["surveillance", "equipment", "due_date"],
                name="unique_execution_per_due_date",
            ),
        ]

    def __str__(self):
        return f"{self.surveillance} on {self.equipment} @ {self.due_date}"
```

The `parameter` FK here directly answers the architecture question from earlier in this project: `core.Surveillance.parameter` is required and can drift if a surveillance is later re-pointed, so the execution snapshots which parameter it was actually checking against, the same way `ParameterReading` snapshots its `value`.

### Step 5.4 — Migration

```bash
python manage.py makemigrations Shiftlog
python manage.py sqlmigrate Shiftlog 0004
python manage.py migrate Shiftlog
```

Confirm the same three things as every earlier migration: only `shiftlog_*` tables, correct `core` dependency, no `core_*` table.

### Step 5.5 — Service

```python
# Shiftlog/services.py — append
from django.utils import timezone

from .models import SurveillanceExecution
from .schedule import advance_next_date


@transaction.atomic
def record_surveillance_execution(*, user, surveillance, result, remarks=""):
    shift = get_current_shift()
    if shift is None:
        raise ValueError("No active shift — cannot record a surveillance execution.")
    if not can_edit_se_log(user, shift):
        raise PermissionError("Only the current responsible SE can record surveillance executions.")

    # select_for_update locks this row until the transaction commits, so two
    # SEs recording the same surveillance at the same instant can't both
    # read the same next_date and both "successfully" advance it —
    # the second one blocks until the first commits, then sees the new value.
    surveillance = type(surveillance).objects.select_for_update().get(pk=surveillance.pk)

    equipment_qs = surveillance.equipment.all()
    if equipment_qs.count() != 1:
        raise ValueError(
            f"Surveillance {surveillance.pk} lists {equipment_qs.count()} equipment; "
            "multi-equipment surveillance is deferred (see Decision 3)."
        )
    equipment = equipment_qs.first()

    now = timezone.now()
    due_date = surveillance.next_date or now.date()

    execution = SurveillanceExecution.objects.create(
        shift=shift,
        area=surveillance.area,
        system=surveillance.system,
        equipment=equipment,
        parameter=surveillance.parameter,
        surveillance=surveillance,
        due_date=due_date,
        executed_at=now,
        executor=user,
        result=result,
        remarks=remarks,
    )

    surveillance.last_date = now.date()
    surveillance.next_date = advance_next_date(surveillance.next_date, surveillance.frequency, now.date())
    surveillance.save(update_fields=["last_date", "next_date", "updated_at"])

    return execution
```

This is the one place in the whole app that writes to a `core` table — exactly as the spec requires, and exactly why it's worth pointing at directly when you re-read the code later.

### Step 5.6 — Selector

```python
# Shiftlog/selectors.py — append
from django.conf import settings as django_settings

from core.models import Surveillance

from .models import SurveillanceExecution
from .schedule import lapses_overdue


def get_surveillance_summary():
    today = timezone.localdate()
    threshold = getattr(django_settings, "SHIFTLOG_LONG_PENDING_LAPSES", 10)

    completed_today = SurveillanceExecution.objects.filter(executed_at__date=today).count()

    due_today = 0
    long_pending = 0
    for surveillance in Surveillance.objects.filter(is_active=True, next_date__isnull=False):
        if surveillance.next_date > today:
            continue
        lapses = lapses_overdue(surveillance.next_date, surveillance.frequency, today)
        if lapses >= threshold:
            long_pending += 1
        else:
            due_today += 1

    return {"completed_today": completed_today, "due_today": due_today, "long_pending": long_pending}
```

This loops in Python rather than a single aggregate query, because "lapses" isn't expressible as plain SQL without a custom date-diff function per frequency. Fine at the scale of a few dozen surveillances; if that count grows into the thousands, this is the function to revisit first, with a raw-SQL `EXTRACT`/`date_part` expression instead of the Python loop.

### Step 5.7 — Template and CSS

```html
{# Shiftlog/templates/Shiftlog/partials/surveillance_status.html #}
<section class="surveillance-summary">
  <div class="surveillance-summary__stat">
    <span class="surveillance-summary__count">{{ surveillance_summary.completed_today }}</span>
    <span class="surveillance-summary__label">Completed Today</span>
  </div>
  <div class="surveillance-summary__stat">
    <span class="surveillance-summary__count">{{ surveillance_summary.due_today }}</span>
    <span class="surveillance-summary__label">Due Today</span>
  </div>
  <div class="surveillance-summary__stat surveillance-summary__stat--warning">
    <span class="surveillance-summary__count">{{ surveillance_summary.long_pending }}</span>
    <span class="surveillance-summary__label">Long Pending</span>
  </div>
</section>
```

```css
/* Shiftlog/static/Shiftlog/css/components/surveillance-summary.css */
.surveillance-summary {
  display: flex;
  gap: var(--sp-6);
  margin-top: var(--sp-4);
}
.surveillance-summary__stat {
  text-align: center;
}
.surveillance-summary__count {
  display: block;
  font-size: var(--text-2xl);
  font-weight: var(--weight-bold);
}
.surveillance-summary__label {
  font-size: var(--text-sm);
  color: var(--color-text-secondary);
}
.surveillance-summary__stat--warning .surveillance-summary__count {
  color: var(--color-warning);
}
```

Wire into the view and template the same way as Step 4.5.

### Step 5.8 — Tests

```python
# Shiftlog/tests/test_schedule.py
from datetime import date

from django.test import SimpleTestCase

from core.models import Surveillance
from Shiftlog.schedule import advance_next_date, lapses_overdue


class ScheduleTests(SimpleTestCase):
    def test_advance_daily(self):
        self.assertEqual(
            advance_next_date(date(2026, 9, 28), Surveillance.Frequency.DAILY, date(2026, 9, 28)),
            date(2026, 9, 29),
        )

    def test_advance_monthly_handles_month_length(self):
        self.assertEqual(
            advance_next_date(date(2026, 1, 31), Surveillance.Frequency.MONTHLY, date(2026, 1, 31)),
            date(2026, 2, 28),  # relativedelta clamps to the shorter month
        )

    def test_lapses_boundaries(self):
        next_date = date(2026, 9, 1)
        # 9 days overdue on a DAILY item = 9 full missed cycles
        self.assertEqual(lapses_overdue(next_date, Surveillance.Frequency.DAILY, date(2026, 9, 10)), 9)
        self.assertEqual(lapses_overdue(next_date, Surveillance.Frequency.DAILY, date(2026, 9, 11)), 10)
        self.assertEqual(lapses_overdue(next_date, Surveillance.Frequency.DAILY, date(2026, 9, 12)), 11)

    def test_not_yet_due_has_zero_lapses(self):
        self.assertEqual(
            lapses_overdue(date(2026, 9, 30), Surveillance.Frequency.DAILY, date(2026, 9, 28)), 0
        )
```

Add `test_services.py` and `test_selectors.py` cases mirroring Milestone 4's shape: immutability of a recorded execution, the multi-equipment refusal, a duplicate `(surveillance, equipment, due_date)` raising `IntegrityError`, and the bucket counts against a couple of hand-built `Surveillance` rows.

### Step 5.9 — Run, verify, commit

```bash
python manage.py test Shiftlog
```

Before testing by hand, take the restore point if you haven't already (Milestone 0, step 8) — this milestone mutates real `core.Surveillance` rows. Then:

```bash
python manage.py shell
```
```python
from Shiftlog.services import record_surveillance_execution
from core.models import Surveillance
s = Surveillance.objects.filter(equipment__isnull=False).first()
# ... call record_surveillance_execution(user=<your SE>, surveillance=s, result="SATISFACTORY")
```

Reload the dashboard, confirm the three counts. Check `s.refresh_from_db()` shows the advanced `last_date`/`next_date`. Restore the dump afterward if you want the original data back:

```bash
pg_restore --clean --if-exists -d <db> pre_shiftlog.dump
```

```bash
git add Shiftlog mysite/settings.py
git commit -m "feat(shiftlog): surveillance execution, schedule advancement and status"
```

---

## Milestones 6–10 — contracts for what's next

You now have every pattern the rest of the app reuses: `TimeStamped` base, `PROTECT` on every `core` FK, selectors that never mutate, services that are `@transaction.atomic` and re-check permission, and one dashboard partial per feature. The remaining milestones are built the same way. Here's exactly what each needs — ask for full code on any of these when you get there.

**Milestone 6 — SE log and signing**
- `SELog(shift FK PROTECT, author FK user PROTECT, content TextField, status CharField[DRAFT, SIGNED], timestamps)`.
- `services.sign_se_log(user, se_log)`: atomic; requires all five `DASHBOARD_PARAMETERS` to have a `ParameterReading` in the current shift (query via `ParameterReading.objects.filter(shift=..., parameter__in=[...])` and compare counts); raises if incomplete.
- `permissions.can_edit_se_log` already exists (Milestone 3) — reuse it, don't rewrite it.
- Routes: current-shift only, no shift ID in the URL (`selectors.get_current_shift()` decides which one).
- Tests: signing succeeds only when complete; a signed log can't be edited; the six permission scenarios from the original spec §25.

**Milestone 7 — Handover**
- `Handover(outgoing_shift FK, outgoing_se FK, incoming_se FK nullable, status CharField[NOT_STARTED, IN_PROGRESS, COMPLETED, TRANSFERRED], open_events/pending_actions/observations/management_instructions/acknowledgements TextFields, timestamps)`.
- `services.start_handover`, `complete_handover` (requires SE log signed), `transfer_responsibility` — each `@transaction.atomic`, each using `select_for_update()` on the `Shift` and `ShiftAssignment` rows they touch, for the same race-safety reason as Step 5.5.
- An explicit `ALLOWED_TRANSITIONS = {NOT_STARTED: {IN_PROGRESS}, IN_PROGRESS: {COMPLETED}, COMPLETED: {TRANSFERRED}}` dict in `services.py`, checked before every status change.
- Pending surveillance and abnormalities shown in the handover panel are computed by calling `get_surveillance_summary()` again — not stored as a copy.

**Milestone 8 — CR area log**
- `CRAreaLog(shift FK, author FK user, area FK core.Area PROTECT nullable, system FK core.System PROTECT nullable, entry TextField, status CharField, timestamp)`.
- Mirrors `SELog`'s form/service/permission/view shape exactly, with `is_control_room_operator` instead of `is_shift_engineer`.

**Milestone 9 — CSS polish**
- Token audit against `mysite/static/mysite/css/design/tokens.css`; dark mode via the existing `[data-theme="dark"]` selector pattern already in that file; keyboard/focus pass; confirm `git diff --stat main -- UserAuthentication/static` is empty.

**Milestone 10 — Final verification and report**
- `python manage.py check`, `makemigrations --check --dry-run`, `migrate --plan`, full `python manage.py test`, compared against the Milestone 0 baseline count.
- `sqlmigrate` every Shiftlog migration once more, confirming none ever touched a `core_*` table.
- A `Shiftlog/README.md` documenting the six decisions, the Trips & Shutdowns data-note, and the deferred multi-equipment case.
- The spec's §28 final report, ending with the explicit line: *db.sqlite3 was NOT used as the runtime database.*

---

## Cheat sheet

| Purpose | Command |
|---|---|
| Health check | `python manage.py check` |
| Migration drift | `python manage.py makemigrations --check --dry-run` |
| Create migrations | `python manage.py makemigrations Shiftlog` |
| Inspect SQL before applying | `python manage.py sqlmigrate Shiftlog <number>` |
| Apply | `python manage.py migrate Shiftlog` |
| Tests (app) | `python manage.py test Shiftlog -v 2 --keepdb` |
| Tests (all) | `python manage.py test --keepdb` |
| Auth migration name (for the groups data migration) | `python manage.py showmigrations auth` |
| Restore point | `pg_dump -Fc -h <host> -U <user> -d <db> -f pre_shiftlog.dump` |
| Restore | `pg_restore --clean --if-exists -d <db> pre_shiftlog.dump` |
