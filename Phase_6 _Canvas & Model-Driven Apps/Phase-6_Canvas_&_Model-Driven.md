
### Phase Objective

Phase 6 delivers the operational application layer of the **Intelligent Equipment Operations Hub** using Microsoft Power Apps and Dataverse.

The objective was to provide technicians with a practical application for viewing equipment information, reporting and updating operational issues, creating and managing maintenance actions, and progressing work through controlled operational status lifecycles.

The resulting Canvas App is named:

**Equipment Technician Hub**

---

## 1. Technology Stack

| Component        | Technology                                 |
| ---------------- | ------------------------------------------ |
| Application      | Microsoft Power Apps Canvas App            |
| Operational Data | Microsoft Dataverse                        |
| Data Tables      | Sites, Assets, Issues, Maintenance Actions |
| App Logic        | Power Fx                                   |
| Analytics Layer  | Microsoft Fabric + Power BI                |
| Application Type | Technician operational application         |

---

## 2. Dataverse Operational Model

The Canvas App uses four Dataverse tables created in the previous phase.

### Sites

Stores site-level operational information.

Key fields include:

- Site_ID
- Site Name
- Location
- Region
- Country
- Site Type

### Assets

Stores equipment and asset information.

Key fields include:

- Asset_ID
- Asset Name
- Asset Type
- Site
- Criticality
- Asset Status
- Health Score

### Issues

Stores equipment failures and operational issues.

Key fields include:

- Issue_ID
- Name
- Issue Title
- Asset
- Severity
- Issue Status
- Reported Date
- Detection Method
- Production Impact
- Description
- Root Cause

### Maintenance Actions

Stores maintenance activities associated with operational issues.

Key fields include:

- Action_ID
- Name
- Action Title
- Issue
- Action Type
- Action Status
- Scheduled Date
- Started Date
- Completed Date
- Assigned To
- Estimated Cost
- Actual Cost
- Notes

---

## 3. Canvas App Architecture

The application follows a simple technician-focused navigation model.

```text
Equipment Technician Hub
        |
        +-- Recent Issues
        |      +-- Issue Detail
        |      +-- Edit Issue
        |      +-- Report New Issue
        |
        +-- Maintenance Actions
        |      +-- Maintenance Detail
        |      +-- Edit Maintenance
        |      +-- Create Maintenance Action
        |
        +-- Assets
        |      +-- Asset Detail
        |
        +-- Sites
               +-- Site Detail
```

The design intentionally keeps the operational workflow lean while maintaining direct integration with Dataverse.

---

## 4. Technician Home Screen

**Screen:** `scrTechnicianHome`

The Technician Home screen acts as the main operational dashboard.

It contains four galleries:

- Recent Issues
- Maintenance Actions
- Assets
- Sites

The screen also provides actions for:

- Reporting a new issue
- Creating a maintenance action
- Editing a selected issue
- Editing a selected maintenance action
- Refreshing operational data

### Automatic Data Refresh

The Home screen refreshes the main Dataverse sources when required:

```powerfx
Refresh(Sites);
Refresh(Assets);
Refresh(Issues);
Refresh('Maintenance Actions')
```

A manual refresh control was also implemented so technicians can request the latest Dataverse data.

---

## 5. Issue Detail Workflow

**Screen:** `scrIssueDetail`
**Form:** `frmIssueDetail`

The selected issue is stored in:

```powerfx
varSelectedIssue
```

The form uses:

```powerfx
Item = varSelectedIssue
```

### Issue Selection

A selected issue is captured before navigating to its detail screen.

```powerfx
Set(varSelectedIssue, ThisItem);
Navigate(scrIssueDetail, ScreenTransition.Fade)
```

### Back Navigation

```powerfx
Back()
```

---

## 6. Report New Issue

A dedicated New Issue screen allows technicians to create operational issues directly in Dataverse.

The form captures information such as:

- Issue ID
- Issue Title
- Asset
- Severity
- Reported Date
- Detection Method
- Production Impact
- Description
- Root Cause
- Issue Status

### Required Dataverse Name Field

Dataverse requires the primary `Name` field.

To avoid duplicate manual entry, the Name field is automatically populated from Issue Title.

Example:

```powerfx
DataCardValue4.Default = DataCardValue3.Text
```

The Name DataCard submits:

```powerfx
DataCardValue4.Text
```

### Submission

```powerfx
SubmitForm(frmNewIssue)
```

The workflow was validated by confirming newly submitted records directly in Dataverse.

---

## 7. Edit Issue Workflow

From the Home screen, the selected issue is captured and the Issue Detail form is opened in Edit mode.

```powerfx
Set(varSelectedIssue, galRecentIssues.Selected);
EditForm(frmIssueDetail);
Navigate(scrIssueDetail, ScreenTransition.Fade)
```

Changes are saved using:

```powerfx
SubmitForm(frmIssueDetail)
```

### Successful Update

```powerfx
Notify(
    "Issue updated successfully.",
    NotificationType.Success
);
Refresh(Issues);
ViewForm(frmIssueDetail)
```

This returns the form to read-only mode after a successful Dataverse update.

---

## 8. Issue Status Lifecycle

The application supports direct issue lifecycle management.

```text
Open <-> In Progress <-> Resolved
```

### Mark In Progress

```powerfx
Patch(
    Issues,
    varSelectedIssue,
    {
        'Issue Status': 'Issue Status (Issues)'.'In Progress'
    }
);

Refresh(Issues);

Set(
    varSelectedIssue,
    LookUp(
        Issues,
        Issue_ID = varSelectedIssue.Issue_ID
    )
);

Notify(
    "Issue status updated to In Progress.",
    NotificationType.Success
)
```

### Mark Resolved

```powerfx
Patch(
    Issues,
    varSelectedIssue,
    {
        'Issue Status': 'Issue Status (Issues)'.Resolved
    }
);

Refresh(Issues);

Set(
    varSelectedIssue,
    LookUp(
        Issues,
        Issue_ID = varSelectedIssue.Issue_ID
    )
);

Notify(
    "Issue status updated to Resolved.",
    NotificationType.Success
)
```

### Mark Open

```powerfx
Patch(
    Issues,
    varSelectedIssue,
    {
        'Issue Status': 'Issue Status (Issues)'.Open
    }
);

Refresh(Issues);

Set(
    varSelectedIssue,
    LookUp(
        Issues,
        Issue_ID = varSelectedIssue.Issue_ID
    )
);

Notify(
    "Issue status updated to Open.",
    NotificationType.Success
)
```

### Context-Aware Buttons

The button representing the current status is hidden.

Example:

```powerfx
varSelectedIssue.'Issue Status' <>
    'Issue Status (Issues)'.'In Progress'
```

This prevents redundant status actions and keeps the interface cleaner.

---

## 9. Create Maintenance Action

A dedicated maintenance creation screen allows technicians to create work directly against operational issues.

**Form:** `frmNewMaintenance`

The form includes:

- Action ID
- Action Title
- Issue
- Action Type
- Action Status
- Scheduled Date
- Assigned To
- Estimated Cost
- Notes
- Name

### New Form Navigation

```powerfx
NewForm(frmNewMaintenance);
Navigate(scrNewMaintenance, ScreenTransition.Fade)
```

### Default Maintenance Status

New maintenance actions default to Planned.

```powerfx
['Action Status (Maintenance Actions)'.Planned]
```

### Scheduled Date

```powerfx
Today()
```

### Required Name Field

The Dataverse primary Name field is populated automatically from Action Title.

```powerfx
DataCardValue20.Default = DataCardValue9.Text
```

The Name DataCard Update property uses:

```powerfx
DataCardValue20.Text
```

### Submission

```powerfx
SubmitForm(frmNewMaintenance)
```

### Successful Creation

```powerfx
Notify(
    "Maintenance action created successfully.",
    NotificationType.Success
);
Refresh('Maintenance Actions');
ResetForm(frmNewMaintenance);
Navigate(scrTechnicianHome, ScreenTransition.Fade)
```

---

## 10. Dataverse Lookup Handling

### Issue Lookup

The Maintenance Action Issue lookup was configured to display the Issue Title rather than generic values such as `Item 1`.

This allows technicians to select meaningful records such as:

```text
FLR0001 - Electrical Fault - AST0005
```

### Assigned To Lookup

The Assigned To field uses the Dataverse Users table.

Enabled users were filtered using:

```powerfx
Filter(
    Users,
    'Status' = 'Status (Users)'.Enabled &&
    !StartsWith('Full Name', "#")
)
```

This reduced the number of system/application identities displayed in the selector.

---

## 11. Maintenance Detail and Edit Workflow

**Screen:** `scrMaintenanceDetail`
**Form:** `frmMaintenanceDetail`

The selected record is stored in:

```powerfx
varSelectedMaintenance
```

The form Item property uses:

```powerfx
varSelectedMaintenance
```

### Edit Maintenance

```powerfx
Set(varSelectedMaintenance, galMaintenanceActions.Selected);
Set(varMaintenanceEditMode, true);
Navigate(scrMaintenanceDetail, ScreenTransition.Fade);
EditForm(frmMaintenanceDetail)
```

An explicit edit-state variable was introduced because relying only on the form Mode property did not consistently control Save button visibility.

### Save Changes

```powerfx
SubmitForm(frmMaintenanceDetail)
```

Save button visibility:

```powerfx
varMaintenanceEditMode
```

### Successful Update

```powerfx
Notify(
    "Maintenance action updated successfully.",
    NotificationType.Success
);
Refresh('Maintenance Actions');
Set(varMaintenanceEditMode, false);
ViewForm(frmMaintenanceDetail)
```

---

## 12. Maintenance Lifecycle

The completed maintenance lifecycle is:

```text
Planned -> Assigned -> In Progress -> Completed
                     \
                      -> Cancelled
```

Cancellation is also available from appropriate pre-completion states.

### Mark Assigned

```powerfx
Patch(
    'Maintenance Actions',
    varSelectedMaintenance,
    {
        'Action Status': 'Action Status (Maintenance Actions)'.Assigned
    }
);

Refresh('Maintenance Actions');

Set(
    varSelectedMaintenance,
    LookUp(
        'Maintenance Actions',
        Action_ID = varSelectedMaintenance.Action_ID
    )
);

Notify(
    "Maintenance action marked as Assigned.",
    NotificationType.Success
)
```

Visible only when:

```powerfx
varSelectedMaintenance.'Action Status' =
    'Action Status (Maintenance Actions)'.Planned
```

### Start Maintenance

```powerfx
Patch(
    'Maintenance Actions',
    varSelectedMaintenance,
    {
        'Action Status': 'Action Status (Maintenance Actions)'.'In Progress'
    }
);

Refresh('Maintenance Actions');

Set(
    varSelectedMaintenance,
    LookUp(
        'Maintenance Actions',
        Action_ID = varSelectedMaintenance.Action_ID
    )
);

Notify(
    "Maintenance started successfully.",
    NotificationType.Success
)
```

The action is hidden for In Progress, Completed and Cancelled records.

### Complete Maintenance

Completing maintenance updates both the status and completion timestamp.

```powerfx
Patch(
    'Maintenance Actions',
    varSelectedMaintenance,
    {
        'Action Status': 'Action Status (Maintenance Actions)'.Completed,
        'Completed Date': Now()
    }
);

Refresh('Maintenance Actions');

Set(
    varSelectedMaintenance,
    LookUp(
        'Maintenance Actions',
        Action_ID = varSelectedMaintenance.Action_ID
    )
);

Notify(
    "Maintenance completed successfully.",
    NotificationType.Success
)
```

The Complete action is shown only when:

```powerfx
varSelectedMaintenance.'Action Status' =
    'Action Status (Maintenance Actions)'.'In Progress'
```

### Cancel Maintenance

```powerfx
Patch(
    'Maintenance Actions',
    varSelectedMaintenance,
    {
        'Action Status': 'Action Status (Maintenance Actions)'.Cancelled
    }
);

Refresh('Maintenance Actions');

Set(
    varSelectedMaintenance,
    LookUp(
        'Maintenance Actions',
        Action_ID = varSelectedMaintenance.Action_ID
    )
);

Notify(
    "Maintenance action cancelled.",
    NotificationType.Success
)
```

Cancellation is hidden for Completed and already Cancelled actions.

---

## 13. Completed Date Handling

A blank Completed Date originally displayed an inappropriate historical/default date.

The Completed Date control was updated so an unfinished maintenance action displays no completion date.

```powerfx
If(
    IsBlank(Parent.Default),
    Blank(),
    Parent.Default
)
```

Completed actions continue to display the actual Dataverse completion timestamp.

---

## 14. Asset and Site Drill-Through

The Home screen provides read-only navigation to Asset and Site details.

### Asset

```powerfx
Set(varSelectedAsset, ThisItem);
Navigate(scrAssetDetail, ScreenTransition.Fade)
```

The Asset form uses:

```powerfx
Item = varSelectedAsset
```

### Site

```powerfx
Set(varSelectedSite, ThisItem);
Navigate(scrSiteDetail, ScreenTransition.Fade)
```

The Site form uses:

```powerfx
Item = varSelectedSite
```

This provides technicians with supporting operational context without allowing unnecessary modification of master data.

---

## 15. Key Power Fx Variables

| Variable                   | Purpose                                                     |
| -------------------------- | ----------------------------------------------------------- |
| `varSelectedIssue`       | Holds the Issue selected for viewing/editing/status updates |
| `varSelectedMaintenance` | Holds the selected Maintenance Action                       |
| `varSelectedAsset`       | Holds the selected Asset                                    |
| `varSelectedSite`        | Holds the selected Site                                     |
| `varMaintenanceEditMode` | Controls Maintenance edit-state UI behaviour                |

---

## 16. Challenges and Resolutions

### Dataverse Primary Name Requirement

**Problem:** New Issue and Maintenance submissions failed because Dataverse required the primary `Name` column.

**Resolution:** Added the Name field to both forms and automatically populated it from the corresponding title field.

### Lookup Display Values

**Problem:** Lookup controls displayed generic values such as `Item 1`.

**Resolution:** Configured meaningful primary display fields, including Issue Title and User Full Name.

### Choice Field Handling

**Problem:** Dataverse Choice fields required typed Choice values rather than plain text.

**Resolution:** Used Dataverse Choice enumerations directly, for example:

```powerfx
'Action Status (Maintenance Actions)'.Planned
```

### Maintenance Save Button Visibility

**Problem:** `frmMaintenanceDetail.Mode = FormMode.Edit` did not reliably expose the Save Changes button during the navigation workflow.

**Resolution:** Introduced:

```powerfx
varMaintenanceEditMode
```

The variable is set when entering Edit mode and reset after a successful save.

### Blank Completed Date

**Problem:** Incomplete maintenance actions displayed a misleading default date.

**Resolution:** Added explicit blank-date handling to the Completed Date control.

---

## 17. Validation and Testing

End-to-end functional testing was completed against Dataverse.

### Issue Validation

Validated:

- New Issue creation
- Asset lookup
- Issue status selection
- Dataverse submission
- Issue detail navigation
- Edit and Save
- Open status
- In Progress status
- Resolved status
- Reopening/status changes
- Dynamic status-button visibility
- Home gallery refresh
- Back navigation

**Result: PASS**

### Maintenance Validation

Validated:

- New Maintenance creation
- Issue lookup
- Assigned To lookup
- Planned default status
- Scheduled Date default
- Dataverse submission
- Maintenance detail navigation
- Edit and Save
- Planned -> Assigned
- Assigned -> In Progress
- In Progress -> Completed
- Automatic Completed Date
- Cancellation workflow
- Dynamic lifecycle-button visibility
- Save button edit-state visibility
- Home gallery refresh
- Back navigation

**Result: PASS**

---

## 18. Final Deliverables

Phase 6 delivers:

1. **Equipment Technician Hub Canvas App**
2. Dataverse-connected operational galleries
3. Site and Asset drill-through
4. New Issue reporting
5. Issue editing and lifecycle management
6. Maintenance Action creation
7. Maintenance editing and lifecycle management
8. Dataverse lookup integration
9. Context-aware Power Fx actions
10. Automatic completion timestamping
11. Data refresh handling
12. Success notifications and controlled form states
13. End-to-end operational validation

---

## 19. Business Value

The Technician Hub converts the Dataverse operational model into an actionable front-end application.

It enables technicians to:

- Access current operational information from one interface
- Report equipment issues directly into Dataverse
- Track issue resolution status
- Create maintenance actions against issues
- Assign and progress maintenance work
- Record completion automatically
- Cancel maintenance when required
- View supporting asset and site context
- Maintain synchronized operational records for downstream reporting

The application therefore closes the gap between the analytics layer and frontline operational execution.

---

## 20. Phase Completion Status

### Phase 6 — Power Apps Operational Application

**Status: COMPLETE**

The Canvas App has been built, connected to Dataverse, functionally tested, and validated across the core Issue and Maintenance workflows.

The Intelligent Equipment Operations Hub now contains an operational application layer capable of capturing and managing frontline equipment operations while maintaining structured Dataverse records for reporting and future automation/AI capabilities.
"""

output = "/mnt/data/Phase_6.md"
pypandoc.convert_text(content, 'md', format='md', outputfile=output, extra_args=['--standalone'])
print(output)
