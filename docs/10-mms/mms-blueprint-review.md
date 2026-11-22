# MMS blueprint review

| Field | Value |
| --- | --- |
| Pull request | [FusionIIIT/Fusion-Integrated#594](https://github.com/FusionIIIT/Fusion-Integrated/pull/594) |
| Reviewer | Yash Kumar |
| Commit reviewed | `b09b953` — "MMS: draft the module blueprint for review" |
| Branch and base | `SinCosMay:module/mms` from `5595be6` |
| Files in the diff | `docs/10-mms/mms-blueprint.md`, 505 lines added, nothing modified |
| Scope | The Step 3.1–3.8 blueprint gate. There is no application code, migration, test or route in this pull request, so the nine-dimension code review has not been run and cannot be scored yet. |
| Verdict | **Not ready to approve.** 1 blocker, 4 majors, 4 minors. |

This review asks one question: if a builder starts Step 4 from this document tomorrow, does it
tell them something true and buildable? Two of the findings are things that would turn `make
check` red or leave a screen unreachable, so they have to settle before the blueprint is
approved.

---

## 1. What was checked, and how

Two kinds of check. First, the blueprint's own claims against each other: every live id has a
row, no id is invented, every permission is well-formed, every grant points at a permission that
exists, every section-E reference resolves. That is mechanical, so it was done by a script and
the script was run; its output is in §5 rather than a description of it.

Second, the claims that are about this platform rather than about the document: icon names, path
shape, status codes, the permission manifest's own gate, the email transport, timezone, and
whether the database extension the design leans on is present. Each of those was read out of the
repository at the commit above, and each finding below names the file and line it rests on.

What this review did **not** get: the source specification itself. The Problem Statement v1.0,
the v1.4 Requirements Registry, the Use Case, Business Rule and Workflow specifications and the
Traceability Matrix are not in this repository and not in this pull request. So the review can
say the blueprint is internally consistent and consistent with the platform. It cannot say the
blueprint's reading of any rule is the right reading, and one finding below exists precisely
because the document claims a verification that left no artefact behind.

---

## 2. Findings

### B1-01 — Blocker — two permissions are granted to nobody, and the section that says otherwise contradicts itself

`docs/10-mms/mms-blueprint.md:409` (the "Confirmed Admin designation" sentence) and the
`ROLE_GRANTS` block at `docs/10-mms/mms-blueprint.md:383`.

`PERMISSIONS` declares 37 codes. Thirty-five are reachable: 29 are held by `student`,
`mess_manager` or `mess_warden`, and six are listed in `SYSTEM_PERMISSIONS`. Two —
`mms.assignment.view` and `mms.assignment.assign` — are neither held by a designation nor
declared system-only. The script in §5 finds exactly this and nothing else.

This is not a stylistic gap. The manifest command refuses it:

> `code: <permission> is declared but no designation holds it, so the endpoint guarding on it is
> unreachable. Grant it in ROLE_GRANTS, or list it in SYSTEM_PERMISSIONS if only a service
> performs it.`
> — `modules/accesscontrol/management/commands/permission_manifest.py:99-105`

Since `make check` runs `permission_manifest --check`, building §D as written makes the gate red.
The consequence runs further than the gate. `module_grants` is **derived** from `ROLE_GRANTS` and
never listed separately (`permission_manifest.py:56-63`: "A designation holding any of a
module's permissions must be able to enter the module"). With no Admin key in `ROLE_GRANTS`,
nobody can enter `/mess/assignments` to assign a Caretaker at all, so `MMS-UC-001`,
`AFR-MMS-002` and `AFR-MMS-003` have no reachable implementation. The nav row at
`docs/10-mms/mms-blueprint.md:455` is guarded by `mms.assignment.view`, which is the exact case
`ops/checks/nav_matches_routes.py:82-90` calls "a nav item guarded by an unholdable permission is
invisible forever."

The document also disagrees with itself about why. Line 315 says "The Mess Admin designation
remains open … the final registry can be validated and seeded only after that mapping is known."
Line 409 begins "Confirmed Admin designation to receive: …" and then names no designation. One
of those sentences has to go.

**Reproduce.** `python ../mms-blueprint-checks.py`, or read §D and run `permission_manifest
--check` on the module once it exists.

**Done when** the exact ERP designation for the Mess Admin is written into `ROLE_GRANTS` with
those two permissions, and the spelling comes from the fixture rather than from this document —
or, if the designation genuinely is not settled, line 409 is deleted, the two permissions are
moved out of the seeded manifest path, and §H says plainly that no one can perform an assignment
until the designation is named. Either answer is approvable. Two answers, one blocker and one
"confirmed", are not.

### B1-02 — Major — the registry icon is a Font Awesome name

`docs/10-mms/mms-blueprint.md:325`, `"icon": "FaClipboardList"`, in the `MODULE` dict that §D
presents as the exact `registry.py`.

Every icon in this repository's own registry is a Phosphor name —
`modules/placement/registry.py:5` is `"icon": "Briefcase"`, and its nav items use `Checks`,
`ClipboardText`, `UserCheck`, `Megaphone`, `ChartBar`. The review rules name this case directly:
"An `Fa…` icon name in `registry.py` is a finding" (`docs/08-delivery/module-review-prompt.md`,
§7). `nav_icon_parity` resolves the committed icon through `ICON_REGISTRY`, so the wrong family is
a nav that does not draw rather than a cosmetic nit.

Note the narrower point, because it is easy to over-correct: the Font Awesome name **is right** at
line 450, which is the handoff entry for the external `Fusion-client` sidebar — that client's
contract (`docs/05-frontend/navigation-contract.md:41-58`) is Font Awesome. Only the
`registry.py` copy at line 325 has to change.

**Done when** §D's `MODULE["icon"]` is a Phosphor name that exists in the client's `ICON_REGISTRY`,
and §E's handoff keeps its Font Awesome name.

### B1-03 — Major — every endpoint in §E carries a trailing slash, which this platform does not serve

`docs/10-mms/mms-blueprint.md:413` and the table under it (`417-448`): `employees/`,
`assignments/`, `registration-windows/`, `requests/`, `menus/`, `menu-items/{id}/`, `bills/`,
`bills/{id}/`, `balances/`, `feedback/`, `rebates/`, `surveys/`, `special-food-requests/`,
`announcements/`, `operational-reports/{kind}/`.

The rule is explicit and is one of the few the platform picked a side on:

> **No trailing slash.** `APPEND_SLASH = False`. The legacy monolith is inconsistent about this
> and it causes real 404s; we pick one.
> — `docs/01-architecture/api-conventions.md:38`

The reference module agrees: `modules/placement/api/urls.py` registers `postings`,
`postings/<int:pk>`, `postings/<int:pk>/publish`, with no trailing slash anywhere. A builder
copying §E would ship 30 endpoints that 404 on the URL the document printed, and the OpenAPI
schema, the client's one hook per endpoint and every spec test would inherit the mistake.

The action sub-resource naming is a separate, smaller problem — see B1-09.

**Done when** each path is written without a trailing slash (`requests`, `requests/{id}`,
`bills/{id}`) so §E can be pasted into `api/urls.py` unchanged.

### B1-04 — Major — the invariant the whole design rests on needs a database extension nobody has confirmed

`docs/10-mms/mms-blueprint.md:279` and `:503`; §B rows for `OperationalDutySlot`, `Membership`
and `RebateRequest`.

`BR-MMS-003` and `BR-MMS-004` ("exactly one active Mess Caretaker / Warden at a time") are carried
by a PostgreSQL exclusion constraint over a `tsrange`, which needs `btree_gist`. So are the
non-overlapping membership intervals and the rebate overlap rule. The document knows:
"Confirm the extension can be provisioned before writing those migrations."

It is not provisioned anywhere today, and there is no precedent to copy. `btree_gist` and
`ExclusionConstraint` appear in this repository exactly twice, both in this blueprint — no
migration, no model, no `RunSQL` in the tree uses either. `docker-compose.yml:6` runs
`postgres:16-alpine` and creates the application role itself; `CREATE EXTENSION` for
`btree_gist` needs a superuser or a database owner with the right privilege, which is a deploy
question this repository does not answer. Django's `ExclusionConstraint` is available (the
dependency set has DRF ≥ 3.15 and Django ≥ 4.x), so the only open part is the extension.

This is a Major rather than a Blocker because §H lists it honestly, and Step 3.8 exists for open
questions. It is a Major rather than a Minor because the answer changes the data model, and Step
3.2 requires the constraints to be named, not deferred: if the extension is unavailable, the
duty-slot table needs a different shape (a partial unique index on the open row per slot kind plus
`SELECT … FOR UPDATE` on the slot), and every service that locks a slot changes with it.

**Done when** either a verified provisioning path is recorded (which role creates the extension,
in which environment, and where that was run), or the fallback is chosen and §B states the
constraint that replaces the exclusion. "Confirm before writing the migrations" is not a design;
it is a task nobody owns.

### B1-05 — Major — a verification is claimed with no artefact behind it

`docs/10-mms/mms-blueprint.md:315` and `:494`: "The synthetic fixture contains the exact
designation codes mess_manager … and mess_warden" and "The fixture verifies mess_manager and
mess_warden", plus "the public development fixture with IAM revision
9eef142b6eda23972e156538083713ff9f7d3ad6" in the header table.

Neither the fixture dump nor the IAM checkout is in this repository, so nothing in this pull
request lets a reader check that sentence — and the sentence says the check was done. The
designation name is the join key between the manifest and the IAM, so the difference between
`mess_manager` and `Mess Manager` is not a typo, it is a grant that reaches nobody:

> `ROLE_GRANTS` is keyed by designation names exactly as `globals_designation` spells them — they
> are the join key, so a typo is a grant that silently reaches nobody.
> — `docs/08-delivery/new-module-prompt.md`, Step 3.4

**Done when** the document carries the evidence instead of the conclusion: the one-line output of
the query or command that listed those designations, with the IAM revision it ran against, or the
sentence is reworded to "must be confirmed against the fixture" and moved into §H with the rest
of the open questions.

### B1-06 — Minor — §G gives cadences but not the beat entries the platform merges

`docs/10-mms/mms-blueprint.md:478-490`.

Step 3.7 asks for "each beat entry, its cadence, and what breaks". The cadence and the
consequence are there; the entry is not. What `config/celery.py` merges is a module's
`schedule.py`, and `modules/placement/schedule.py` shows what one has to contain: `BEAT_SCHEDULE`
with a named key, a `task` string, a `schedule`, and `options: {"expires": …}` — plus
`TASK_ROUTES` so a drain task does not queue behind a report. Six tasks without keys, task names
or expires values cannot be registered or tested.

**Done when** each §G row names its beat key, task name, interval, `expires` and queue.

### B1-07 — Minor — §D omits `NAV_ITEMS`, so the 13 routes have no sidebar contract

`docs/10-mms/mms-blueprint.md:383-408` gives `MODULE`, `PERMISSIONS`, `SYSTEM_PERMISSIONS`,
`SCOPE_PERMISSIONS` and `ROLE_GRANTS`. Line 450 asserts "Each child route receives a nav item with
the same permission" and the table at 452-467 gives a label and a guard per route — but no item
`code`, `icon` or `sort_order`.

Three consumers need those fields: `permission_manifest._nav` copies `code`, `label`, `icon`,
`to`, `required_permission`, `sort_order` into the seeded nav; `ops/checks/nav_matches_routes.py`
needs `code` and `to`; and the client filters on `required_permission`. Step 3.5 also asks for the
sidebar entry "ready to paste".

**Done when** §D or §E lists the 13 nav items with all six fields, in the order the sidebar
should draw them.

### B1-08 — Minor — enforcement points name no place in the code

`docs/10-mms/mms-blueprint.md:11-24` and every section-A table: "MMS model, route and integration
inventory", "assign operational duty", "read feedback".

Step 3.1's fourth column is the column the later review uses — "Every live BR, UC and workflow id
from your own inventory is implemented at the enforcement point the blueprint names. Open that
file and read the code" (`module-review-prompt.md`, §3, dimension 1). A phrase like "assign
operational duty" cannot be opened. The 165 repeated cells are also where a copy-paste error
would hide: `AFR-MMS-029` to `031` all say "maintain and publish menu items" whether the row is
add, delete or modify.

**Done when** each row names the layer directory and the service, selector or model it expects
(`services/assignments.py`, `selectors/requests.py`), or §A states that exact paths are filled in
at Step 4 and the reviewer should check the behaviour instead.

### B1-09 — Minor — action sub-resources are named as nouns

`docs/10-mms/mms-blueprint.md:423`, `:429`, `:430`, `:441`: `requests/{id}/decision/`,
`bills/{id}/finalization/`, `bills/{id}/publication/`, `special-food-requests/{id}/fulfillment/`.

The platform's convention for a genuine non-CRUD action is a `POST` to a sub-resource named for
the action: `POST /applications/{id}/withdraw`, `POST /offers/{id}/accept`
(`docs/01-architecture/api-conventions.md:35-37`). The reference module follows it — `publish`,
`transition`, `respond`, `withdraw` in `modules/placement/api/urls.py`. `decision`, `finalization`,
`publication` and `fulfillment` read as resources instead, and they force the permission names to
be chosen around them (`mms.bill.approve` guarding a `finalization` sub-resource is hard to
follow).

**Done when** the four are `decide` (or `decision` if it is kept deliberately and said so),
`finalize`, `publish` and `fulfill`, matching the verb the corresponding permission uses.

---

## 3. What the blueprint gets right

Worth recording, because a list of findings alone misrepresents the document. Each of these was
checked and holds:

- **The id work is complete and honest.** 41 business rules spanning 001–042 with 037 genuinely
  absent, 31 use cases, 22 coherent requirements, 101 atomic requirements, 16 basis and 5
  composite workflows — no duplicate rows, no gaps other than the retirement the header declares.
- **Every permission is legal.** All 37 codes are three `snake_case` segments, the first segment
  equals `MODULE["code"]`, and every verb is inside the closed vocabulary
  (`rbac-model.md:96-101`). `SYSTEM_PERMISSIONS` is a subset of `PERMISSIONS` and no designation
  holds a system-only permission.
- **Section E is wired to the rules.** All 31 permissions it names are declared, every non-boundary
  business rule reaches at least one endpoint, and the five `ISS-MMS-*` ids it cites are exactly
  the five §H rules on.
- **The HTTP semantics match the platform.** 404 for out-of-visibility rows, 403 for a missing
  permission, 422 for semantic refusal, 409 for a conflicting transition, 201 with `Location`,
  cursor pagination — all as `api-conventions.md` states them.
- **Time is handled correctly.** `Asia/Kolkata` is the platform timezone
  (`config/settings/base.py:240-242`); half-open intervals with the successor starting at the
  instant the predecessor ends is what makes "exactly one active" true at the handover instant,
  and it is the right reading of `BR-MMS-006`.
- **The notification design matches the reference module.** `NotificationIntent` with a unique
  event-and-recipient key, written in the same transaction as the state change, is the same shape
  as `modules/placement/services/notifications.py` (`dedupe_key`, outbox in the caller's
  transaction, retry worker with `last_error`). The transport exists: Django's email backend,
  console in development and SMTP in production (`config/settings/base.py:167-168`,
  `config/settings/prod.py:23-24`).
- **Module grants need no separate list.** Deriving them from `ROLE_GRANTS` is what the command
  does, so §D is right not to duplicate it.

---

## 4. Questions for the approver

These rest on the specification, which is not in this repository. None of them lowers a score;
each one changes code if it is answered the other way.

1. **`ISS-MMS-012` (Admin menu access).** The draft withholds `mms.menu.view` from Admin. If the
   ruling goes the other way, `ROLE_GRANTS` and one nav guard change; if it holds, §H should say
   Admin is deliberately excluded rather than left blank.
2. **`ISS-MMS-015` (charge after cancellation).** `BR-MMS-030` charges an approved request;
   `BR-MMS-031` allows Served and Cancelled after Approved. Does a Cancelled request keep its
   charge, drop it, or reverse it — and what happens to a bill already published? The ledger
   design in §B is reversible either way, but the rule is not decidable from the blueprint.
3. **`BR-MMS-037`.** The header says it is retired and the table agrees by omission. Confirm from
   the v1.4 registry that the retirement is real, since a retired rule that is actually live
   would be silently unimplemented.
4. **`BR-MMS-023` (overlapping rebates).** §H leaves open whether a rejected request's interval
   blocks a new one. That decides whether the exclusion constraint covers `Pending` only or
   `Pending` and `Approved`.
5. **Which designation is the Mess Admin**, and may it assign an existing employee who does not
   already hold the designation? This is B1-01's other half.
6. **Served days.** `BR-MMS-017` bills daily rate × served days. In `module-only` there is no
   attendance source; §H says the input is unconfirmed. Who supplies it in the assignment
   environment, and what does a bill do when it is missing for a month?
7. **`BR-MMS-008` and §C disagree about who opens the window.** The rule says only the active
   Caretaker opens it; §C says the window opens when its opening instant arrives. Both can be
   true if "opening" means creating the window, but the blueprint should say so, because one
   reading makes `POST registration-windows` the gate and the other makes the clock the gate.

---

## 5. Verification log

The document checks run from the repository root with the script kept outside the tree, so
`git status` stays clean for a reviewer:

```
$ python ../mms-blueprint-checks.py
BR-MMS: 41 rows, range 001-042, missing numbers [37]
MMS-UC: 31 rows, range 001-031, missing numbers none
CFR-MMS: 22 rows, range 001-022, missing numbers none
AFR-MMS: 101 rows, range 001-101, missing numbers none
workflows: 16 basis + 5 composite = 21
permissions declared: 37
  mms.assignment.assign appears only at line 409 as 'Confirmed Admin …', no designation named
  mms.assignment.view appears only at line 409 as 'Confirmed Admin …', no designation named
section E permissions referenced: 31
client routes: 13
ISS ids used: ['ISS-MMS-012', 'ISS-MMS-013', 'ISS-MMS-014', 'ISS-MMS-015', 'ISS-MMS-019']; ruled in H: [same five]
non-boundary business rules with no endpoint in section E: []

FAILED 2 check(s):
  - permissions no role holds and no task declares: ['mms.assignment.assign', 'mms.assignment.view']
  - line 325: registry.py MODULE icon is a Font Awesome name; the module registry uses Phosphor names
```

Read by hand, at commit `b09b953`:

| Claim in the blueprint | Where the answer lives | Result |
| --- | --- | --- |
| Registry icon | `modules/placement/registry.py:5`; `docs/08-delivery/module-review-prompt.md` §7 | `FaClipboardList` is the wrong family — B1-02 |
| No trailing slash | `docs/01-architecture/api-conventions.md:38`; `modules/placement/api/urls.py` | §E contradicts both — B1-03 |
| Orphan permission fails the gate | `modules/accesscontrol/management/commands/permission_manifest.py:99-105` | fails — B1-01 |
| Module grants come from `ROLE_GRANTS` | same file, lines 56-63 | confirmed, so no separate list is needed |
| Nav item field set | `permission_manifest.py:73-75`; `ops/checks/nav_matches_routes.py` | six fields required, §D has none — B1-07 |
| `btree_gist` / exclusion constraint | grep over the tree: two hits, both this document | no precedent, no provisioning — B1-04 |
| Timezone | `config/settings/base.py:240-242` | `Asia/Kolkata`, and Celery reads it — confirmed |
| Email transport | `config/settings/base.py:167`; `prod.py:23`; `modules/placement/services/notifications.py` | exists, and the outbox shape matches — confirmed |
| Status codes and pagination | `docs/01-architecture/api-conventions.md` §2 | §E agrees — confirmed |
| Beat schedule shape | `config/celery.py`; `modules/placement/schedule.py` | §G lacks entries — B1-06 |

Not run, and why: `make check`, `make check-client` and `make e2e`. This pull request changes one
Markdown file and no code, so those gates have nothing new to judge, and this machine has no
provisioned stack (`.venv`, Postgres, Redis, the IAM on 8001). The document-only checks above are
what this diff can actually fail. Nothing was driven in a browser: there is no page yet.

---

## 6. Run record

| Role | Tool and model | Where |
| --- | --- | --- |
| Reviewer, blueprint gate | GPT-OSS 20B and Qwen 7B | This record |

Sessions: one review pass over `docs/10-mms/mms-blueprint.md` at `b09b953`, plus one scripted
consistency pass over the same commit. No builder session was shared with the review. Web search
was not used, and no other student's pull request was read.

---

## 7. Verdict

`VERDICT: not yet` — 9 open findings:

| # | Severity | Finding |
| --- | --- | --- |
| B1-01 | Blocker | `mms.assignment.view` and `mms.assignment.assign` reach no designation, the manifest gate fails, and §D's "Confirmed Admin" contradicts §D's "remains open" |
| B1-02 | Major | `MODULE["icon"]` is Font Awesome in a Phosphor registry |
| B1-03 | Major | 30 endpoint paths carry a trailing slash against `APPEND_SLASH = False` |
| B1-04 | Major | Exclusion constraints depend on an unconfirmed `btree_gist` |
| B1-05 | Major | Fixture designations and the IAM revision are claimed verified with no evidence |
| B1-06 | Minor | §G has cadences but no beat entries |
| B1-07 | Minor | No `NAV_ITEMS` for the 13 routes |
| B1-08 | Minor | Enforcement points name no file |
| B1-09 | Minor | Action sub-resources are nouns where the platform uses verbs |

Two of these are cheap to fix and change nothing else: B1-02 is one string, and B1-03 is a
find-and-replace plus four renames. B1-01 needs an answer from outside this repository, which is
the one thing worth settling before Step 4 begins — everything else can be corrected in the same
commit.
