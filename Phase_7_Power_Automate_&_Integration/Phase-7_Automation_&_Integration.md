
# Phase 7 — Power Automate Workflows & Approvals


## 1. Objective and scope

Connect the Phase 5–6 Dataverse model and Power Apps experience to an automated operational lifecycle: approve critical issues, create maintenance work without routine duplicates, resolve linked issues on completion, and send new-work and overdue emails.

This is an implementation record of the [Phase-7_Power Automate Workflows &amp; Approvals conversation](chatgpt-conversation://6ab41cb3-f8b8-83ed-b037-b109109d3c90), including its corrections and completion confirmations. It is not a new audit of the live environment. Validation below reflects the conversation; run IDs and exported flow definitions were not supplied.

## 2. Architecture and flow summary

```text
Canvas / operations app → Dataverse Issues
  └─ New Critical issue → [1] Critical Issue Approval
       ├─ Approve → Issue = In Progress
       │    └─ [2] Check generated Action_ID → create Planned maintenance if absent
       └─ Reject → Issue = Closed

Dataverse Maintenance Actions (created by flow or app)
  ├─ Added + Planned → [4] Fetch linked Issue → Gmail notification
  ├─ Modified + Completed → [3] Linked Issue = Resolved
  └─ Daily schedule → [5] Past scheduled date + not Completed → Gmail reminder

Canvas form save → Refresh Maintenance Actions → show newest Created On first
```

All four event flows use **Microsoft Dataverse → When a row is added, modified or deleted**, with **Organization** scope. The fifth uses **Schedule → Recurrence**. The flows were built in the existing solution with Dataverse, Approvals and Gmail connections. Issues link to Assets; Maintenance Actions link to Issues. Site and Asset provide operational context, but these flows do not update them.

Integration is driven by Dataverse events after app saves; no direct Canvas `Flow.Run()` call was documented.

## 3. The five completed flows

### 1 — Critical Issue Approval

- **Trigger:** Added row in **Issues**; optional trigger filters left blank.
- **Condition:** Severity equals **Critical (`100000003`)**. Non-critical branch is empty.
- **Approval:** **Start and wait for an approval**; type **Approve/Reject – First to respond**; title `Critical Issue Approval`. Details contain dynamic Issue_ID, Description, Severity, Asset (Value), and Created On. Item link and description were left blank.
- **Recipient:** A resolvable Microsoft Entra user UPN matching the test approver's signed-in identity. The final working UPN was not transcribed. Approvals were answered through **Power Automate → Approvals → Received**.
- **Outcome condition:** Approval **Outcome = `Approve`**.
- **Inner Yes:** Update the original Issue, using its GUID as Row ID; set **Issue Status = In Progress**.
- **Inner No:** Update the same Issue; set **Issue Status = Closed**. Both updates supply only the status field.

The existing Issue choices were used; separate Approved/Rejected choice values or an Approval Status column were not added. The implemented No branch covers any returned outcome other than `Approve`; an earlier blank outcome also took this branch. Explicit handling of blank/cancelled outcomes was not documented.

### 2 — Create Maintenance Action from Approved Issue

- **Trigger:** Modified row in **Issues**; **Select columns:** `new_issuestatus`; **Filter rows:** blank.
- **Condition:** Issue Status equals **In Progress (`100000003`)**.
- **Duplicate check:** List **Maintenance Actions** filtered by generated `Action_ID = MA-<Issue_ID>`, with **Row count = 1**. Other List rows options remain blank.
- **Nested condition:** `length(body('List_rows')?['value'])` equals `0`.
- **Yes:** Add a Maintenance Action using the mappings below.
- **No:** Leave empty; an existing generated action is retained. The outer false branch is also empty.

| Target field   | Final mapping                                |
| -------------- | -------------------------------------------- |
| Action Status  | Planned (`100000000`)                      |
| Action Type    | Corrective Maintenance (`100000004`)       |
| Action_ID      | `MA-` + Issue business ID                  |
| Action Title   | `Maintenance - ` + Issue Title             |
| Name           | Same expression as Action Title              |
| Issue (Issues) | OData reference to the triggering Issue GUID |

This flow uses **In Progress as the operational signal**. It does not independently verify an approval response or severity: a manual status update to In Progress also activates it. Automatic technician assignment and a scheduled-date default were not part of the recorded mappings.

### 3 — Sync Issue Status from Maintenance

- **Trigger:** Modified row in **Maintenance Actions**; **Select columns:** `new_actionstatus`; **Filter rows:** blank.
- **Condition:** Action Status equals **Completed (`100000003`)**.
- **Yes:** Dataverse **Update a row → Issues**; Row ID = **Issue (Value)** from the maintenance trigger; set **Issue Status = Resolved (`100000001`)**. Leave other fields unpopulated.
- **No:** No action.

The completed implementation synchronizes completion only. It does not synchronize every intermediate state or check whether other maintenance actions linked to the same Issue are still incomplete. A valid Issue lookup is required for the update.

### 4 — Notify Technician of New Maintenance Action

- **Trigger:** Added row in **Maintenance Actions**; Select columns and Filter rows blank.
- **Condition:** Action Status equals **Planned (`100000000`)**.
- **Yes:** **Get a row by ID → Issues**, using **Issue (Value)**; then **Gmail → Send email (V2)**.
- **No:** No action. Assigned-only records were not included in the final condition.
- **To:** Configured test Gmail recipient, `singh.manpreet1308@gmail.com`.
- **Subject:** `New Maintenance Action Assigned`.
- **Body:** Action ID, Action Title, readable Action Type and Action Status, related `Issue_ID - Issue Title` from Get a row by ID, and Scheduled Date. Each label is followed by its actual dynamic token/expression. Attachments were left blank.

Despite the flow name and subject, the implemented recipient is a fixed test inbox; dynamic routing to an assigned technician was not documented.

### 5 — Overdue Maintenance Reminder

- **Trigger:** Scheduled cloud flow; Recurrence **Frequency = Day**, **Interval = 1**. Exact start time and schedule timezone were not recorded.
- **Query:** Dataverse **List rows → Maintenance Actions**, filtered to **Scheduled Date before today's UTC date AND Action Status not Completed**. Select columns, Sort By, Fetch XML and Row count were left blank.
- **Loop:** **Apply to each**, using `value` from List rows.
- **Action:** **Gmail → Send email (V2)** for each returned row, using the configured test Gmail inbox.
- **Subject:** `Overdue Maintenance Reminder - ` + dynamic Action_ID.
- **Body:** Action ID, Action Title, readable Type and Status, Scheduled Date, and `Reminder: This maintenance action is overdue and requires attention.`

The filter excludes today's scheduled work and Completed records. It does **not** exclude Cancelled records. Matching rows can receive another reminder on each daily run; no last-reminded field, suppression window, escalation or reminder deduplication was implemented. A Scheduled Date is needed for a row to match.

## 4. Dataverse names and choice values

Use logical names in expressions and trigger filtering attributes, rather than mixed-case schema names or display labels.

| Object / field              | Recorded logical name or reference                          | Purpose                                       |
| --------------------------- | ----------------------------------------------------------- | --------------------------------------------- |
| Issues table                | `new_issue`; entity set used successfully: `new_issues` | Issue lookup binding                          |
| Maintenance Actions table   | `new_maintenanceaction`                                   | Select the table from the connector dropdown  |
| Issue row GUID              | `new_issueid`                                             | Update Row ID / relationship target           |
| Issue business ID           | `new_issue_id`                                            | Human-facing Issue_ID and generated Action_ID |
| Issue Title                 | `new_issuetitle`                                          | Maintenance title/name                        |
| Issue Status                | `new_issuestatus`                                         | Status condition and selected trigger column  |
| Severity                    | `new_severity`                                            | Critical-issue condition                      |
| Description / Name          | `new_description` / `new_name`                          | Issue details / primary name                  |
| Issue Asset lookup value    | `_new_asset_value`                                        | Asset GUID in approval details                |
| Maintenance Action_ID       | `new_action_id`                                           | Duplicate-check filter                        |
| Action Status / Type        | `new_actionstatus` / `new_actiontype`                   | Conditions and email labels                   |
| Scheduled Date              | `new_scheduleddate`                                       | Overdue query                                 |
| Created On                  | `createdon`                                               | Canvas gallery sort                           |
| Maintenance → Issue lookup | Dynamic token**Issue (Value)**                        | Linked Issue GUID for Get/Update row          |

**Do not confuse `new_issue_id` with `new_issueid`:** the first is a business ID such as `106`; the second is the Dataverse GUID. The raw logical property for the maintenance Issue lookup was not transcribed; the working dynamic token is recorded above.

| Choice field  | Confirmed labels and stored values                                                                                                                                                                                 |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Severity      | Critical =`100000003`                                                                                                                                                                                            |
| Issue Status  | Open =`100000002`; In Progress = `100000003`; Resolved = `100000001`; Closed selected by label, numeric value not transcribed                                                                                |
| Action Status | Planned =`100000000`; Assigned = `100000001`; In Progress = `100000002`; Completed = `100000003`; Cancelled = `100000004`                                                                                |
| Action Type   | Inspection =`100000000`; Repair = `100000001`; Replacement = `100000002`; Preventive Maintenance = `100000003`; Corrective Maintenance = `100000004`; Calibration = `100000005`; Other = `100000006` |

These values belong to the recorded environment. The custom Issue Status is separate from Dataverse system `statecode`/`statuscode`.

## 5. Key expressions and OData filters

Enter expressions through the **Expression** tab so they appear as **fx tokens**, rather than storing literal `concat(...)` text. Action references such as `List_rows` and `Apply_to_each` must match the flow's internal action names.

### Maintenance creation and duplicate prevention

```text
// Action_ID
concat('MA-', triggerOutputs()?['body/new_issue_id'])

// Action Title and Name
concat('Maintenance - ', triggerOutputs()?['body/new_issuetitle'])

// Issue (Issues) lookup binding
concat('/new_issues(', triggerOutputs()?['body/new_issueid'], ')')

// List rows → Filter rows
concat(
  'new_action_id eq ''MA-',
  triggerOutputs()?['body/new_issue_id'],
  ''''
)

// Nested condition: result equals 0
length(body('List_rows')?['value'])
```

The comments label separate expressions; paste only the expression into each field. For Issue_ID `106`, the filter resolves to `new_action_id eq 'MA-106'`.

This prevents repeated creation for an existing generated ID, including when an Issue returns to In Progress. It is an ID-based existence check, not a query for every maintenance row linked to that Issue. A manually created action with a different ID does not block creation. Concurrent-run locking or an alternate key was not documented, so this is not a demonstrated atomic uniqueness guarantee.

### Readable email choices

```text
// New maintenance notification: trigger labels
triggerOutputs()?['body/_new_actiontype_label']
triggerOutputs()?['body/_new_actionstatus_label']

// Overdue reminder: current loop item labels
items('Apply_to_each')?['_new_actiontype_label']
items('Apply_to_each')?['_new_actionstatus_label']
```

These are the expressions used in the recorded setup. Notification formatting was confirmed working. Get the linked Issue separately to display its business ID/title rather than its GUID.

### Overdue filter

```text
concat(
  'new_scheduleddate lt ',
  formatDateTime(utcNow(),'yyyy-MM-dd'),
  ' and new_actionstatus ne 100000003'
)
```

Example resolved filter on 25 September 2026:

```odata
new_scheduleddate lt 2026-09-25 and new_actionstatus ne 100000003
```

This records the working date-only filter used in the project. Its boundary is derived from `utcNow()`, not the Canvas user's local date.

## 6. Canvas App refresh and gallery fix

**Symptom:** A newly saved maintenance action existed in Dataverse but did not appear in the home gallery. Its Items formula selected the first four rows sorted by **Completed Date descending**; new actions normally had no Completed Date.

Final **`frmNewMaintenance.OnSuccess`**:

```powerfx
Notify(
    "Maintenance action created successfully.",
    NotificationType.Success
);
Refresh('Maintenance Actions');
ResetForm(frmNewMaintenance);
Navigate(scrTechnicianHome, ScreenTransition.Fade)
```

Final maintenance gallery **Items**:

```powerfx
FirstN(
    Sort(
        'Maintenance Actions',
        'Created On',
        SortOrder.Descending
    ),
    4
)
```

**Result:** Successful save → refresh Dataverse data → reset form → navigate home → newest maintenance appears at the top. The user confirmed “done now works fine.” The gallery intentionally remains a four-record preview; large-dataset delegation behavior was not tested in this phase.

## 7. Testing and validation outcomes

| Test                                                 | Recorded outcome                                                                                                                                  |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Critical issue, Approve                              | Successful approval branch;`Pump_Issue03` / Issue_ID `103` displayed In Progress in the app                                                   |
| Critical issue, Reject                               | Approval completed, outcome condition false, rejection update executed; final overall validation was subsequently confirmed                       |
| In Progress issue with no generated maintenance      | Creation succeeded;`MA-105` and later clean test `MA-106` showed Planned, Corrective Maintenance, generated title/name and correct Issue link |
| In Progress issue with an existing generated ID      | List rows succeeded; zero-count condition false; Add a new row skipped, preventing a duplicate                                                    |
| Nonmatching Issue status                             | Outer condition false; downstream actions skipped as intended                                                                                     |
| Linked maintenance changed to Completed              | Update succeeded; linked Issue Status became Resolved (`100000001`)                                                                             |
| New Planned maintenance notification                 | Gmail received the email; readable Type/Status and related Issue ID/title confirmed after formatting fixes                                        |
| Reminder with no matching overdue rows               | Successful query, zero loop iterations, email skipped; this was not evidence of email delivery                                                    |
| Reminder with past Scheduled Date and Planned status | Controlled test record produced an overdue reminder; user explicitly confirmed receipt                                                            |
| Final flow review                                    | After the five-flow checklist, user reported “yes all checked and works fine”                                                                   |
| Canvas maintenance creation                          | Refresh and Created On sort applied; user confirmed the new record appeared correctly                                                             |

Flow checker initially reported no errors for the approval flow, and checking all five flows for zero errors, valid connections, precise triggers and successful run history was included in final acceptance. The final confirmation was aggregate; separate checker exports were not captured. The daily reminder was tested manually; an unattended scheduled-run history was not separately supplied.

## 8. Issues encountered and fixes

| Issue                                                                       | Final resolution / lesson                                                                                                    |
| --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Severity compared with text`Critical` returned false                      | Inspect trigger JSON; compare with numeric`100000003`                                                                      |
| Rejection action placed in outer non-critical branch                        | Move Closed update into the inner approval-outcome No branch; remove extra Condition 3                                       |
| `Create an approval` did not provide the intended wait-and-branch pattern | Replace with**Start and wait for an approval**                                                                         |
| Empty/invalid`assignedTo`; request visible in Sent but not Received       | Resolve the actual Entra approver UPN, match signed-in identity, and respond to the correct fresh request in Received        |
| Blank approval outcome followed Closed branch                               | Diagnose approval output and identity/request mismatch; retest with a valid received approval, confirming Approve separately |
| Mixed-case`new_ActionStatus` rejected as filtering attribute              | Use lowercase logical`new_actionstatus`; Issue trigger uses `new_issuestatus`                                            |
| Expressions entered as plain text                                           | Re-enter through Expression tab as fx tokens, including generated IDs/titles and overdue filter                              |
| Raw GUID failed in maintenance Issue lookup with OData path error           | Bind using`/new_issues(<new_issueid GUID>)`                                                                                |
| Copied/rebuilt cards lost connection or dynamic schema                      | Reselect valid Dataverse connection references and restore field mappings                                                    |
| `EntityNotFound` for typed `Maintenance Actions`                        | Recreate action and select table from dropdown so connector metadata resolves                                                |
| Mail notification V3 returned Unauthorized                                  | Replace with authenticated**Gmail → Send email (V2)**                                                                 |
| Email showed numeric choices, GUIDs or literal placeholders                 | Insert actual dynamic tokens, use label expressions and fetch the linked Issue                                               |
| Suggested inline`switch(...)` failed                                      | Replace with recorded`_new_actiontype_label` / `_new_actionstatus_label` expressions                                     |
| Grey/skipped downstream actions looked like failures                        | Inspect branch conditions and loop count; deliberate skips are expected                                                      |
| New action missing from Canvas gallery                                      | Refresh after save and sort the four-row preview by Created On instead of Completed Date                                     |

## 9. Final acceptance checklist and status

- [X] Five named flows created and reviewed in the project solution.
- [X] Critical approval routes Approve to In Progress and Reject to Closed.
- [X] In Progress triggers linked Planned / Corrective Maintenance creation.
- [X] Generated-ID duplicate check validated for both create and skip branches.
- [X] Completed maintenance resolves its linked Issue.
- [X] New Planned maintenance sends a readable Gmail notification.
- [X] Daily overdue query configured; reminder email delivery tested successfully.
- [X] Connection references, trigger settings, flow checking and run history included in the user-confirmed final review.
- [X] Canvas save refreshes Maintenance Actions and shows newest records first.

**Final status: Phase 7 complete for the implemented and tested project scope.** The accepted chain is critical issue → approval → maintenance creation → notification → completion/status synchronization, with scheduled overdue reminders and a working Canvas gallery.

Future hardening, outside this acceptance: dynamic technician routing; atomic duplicate protection; explicit unexpected approval-outcome handling; multiple-maintenance completion rules; cancelled-item reminder policy; and dedicated failure alerts/retry monitoring. These are not claimed as completed features.
