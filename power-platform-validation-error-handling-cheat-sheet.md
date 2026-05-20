# Power Apps & Power Automate Cheat Sheet: Validation, Error Handling & Flow Resilience

> A practical GitHub-ready cheat sheet for building production-ready Power Apps and Power Automate flows.

## Core Principle

Production-ready solutions do not assume everything works.

They:

- prevent bad data before submission
- explain errors clearly to users
- log useful technical context for support
- recover safely where possible
- avoid duplicate or unsafe processing
- make failures visible to owners

---

## Table of Contents

1. [Power Apps Validation](#power-apps-validation)
2. [Power Apps Error Handling](#power-apps-error-handling)
3. [Power Apps User-Friendly Error Messages](#power-apps-user-friendly-error-messages)
4. [Power Automate Try/Catch/Finally Pattern](#power-automate-trycatchfinally-pattern)
5. [Power Automate Flow Expressions](#power-automate-flow-expressions)
6. [Getting Flow Run Details](#getting-flow-run-details)
7. [Getting Error Details from Failed Actions](#getting-error-details-from-failed-actions)
8. [Filtering Failed Results from a Try Scope](#filtering-failed-results-from-a-try-scope)
9. [Returning Errors Back to Power Apps](#returning-errors-back-to-power-apps)
10. [Retry and Resilience Patterns](#retry-and-resilience-patterns)
11. [Production Checklist](#production-checklist)

---

# Power Apps Validation

## Required Field

```powerfx
IsBlank(txtTitle.Text)
```

Example submit guard:

```powerfx
If(
    IsBlank(txtTitle.Text),
    Notify(
        "Enter an incident title before continuing.",
        NotificationType.Error
    ),
    SubmitForm(frmIncident)
)
```

---

## Email Format Validation

```powerfx
!IsMatch(
    txtEmail.Text,
    Match.Email
)
```

Example:

```powerfx
If(
    !IsMatch(txtEmail.Text, Match.Email),
    Notify(
        "Enter a valid email address so we can send the confirmation.",
        NotificationType.Error
    )
)
```

---

## Minimum Length Validation

```powerfx
Len(txtDescription.Text) < 20
```

Example:

```powerfx
If(
    Len(txtDescription.Text) < 20,
    Notify(
        "Add a little more detail so the support team can understand the issue.",
        NotificationType.Warning
    )
)
```

---

## Date Cannot Be in the Future

```powerfx
dpIncidentDate.SelectedDate > Today()
```

Example:

```powerfx
If(
    dpIncidentDate.SelectedDate > Today(),
    Notify(
        "The incident date cannot be in the future.",
        NotificationType.Error
    )
)
```

---

## Disable Submit Until Form Is Valid

```powerfx
If(
    frmIncident.Valid,
    DisplayMode.Edit,
    DisplayMode.Disabled
)
```

For a custom validation approach:

```powerfx
If(
    !IsBlank(txtTitle.Text) &&
    IsMatch(txtEmail.Text, Match.Email) &&
    dpIncidentDate.SelectedDate <= Today() &&
    Len(txtDescription.Text) >= 20,
    DisplayMode.Edit,
    DisplayMode.Disabled
)
```

---

## Validation Summary Pattern

Create a collection of validation issues before submitting:

```powerfx
ClearCollect(
    colValidationErrors,
    If(
        IsBlank(txtTitle.Text),
        { Field: "Title", Message: "Enter an incident title." }
    ),
    If(
        !IsMatch(txtEmail.Text, Match.Email),
        { Field: "Email", Message: "Enter a valid email address." }
    ),
    If(
        dpIncidentDate.SelectedDate > Today(),
        { Field: "Incident Date", Message: "Incident date cannot be in the future." }
    ),
    If(
        Len(txtDescription.Text) < 20,
        { Field: "Description", Message: "Add at least 20 characters of detail." }
    )
);
```

Then only submit when there are no validation errors:

```powerfx
If(
    CountRows(colValidationErrors) > 0,
    Notify(
        "Check the highlighted fields before submitting.",
        NotificationType.Error
    ),
    SubmitForm(frmIncident)
)
```

---

# Power Apps Error Handling

## Basic `IfError` Pattern

Use `IfError` around risky operations such as `Patch`, connector calls, or flow calls.

```powerfx
IfError(
    Patch(
        Incidents,
        Defaults(Incidents),
        {
            Title: txtTitle.Text,
            Description: txtDescription.Text
        }
    ),
    Notify(
        "We couldn't save your incident. Please try again.",
        NotificationType.Error
    )
)
```

---

## `IfError` with `FirstError.Message`

```powerfx
IfError(
    Patch(
        Incidents,
        Defaults(Incidents),
        {
            Title: txtTitle.Text,
            Description: txtDescription.Text
        }
    ),
    Notify(
        "We couldn't save your incident. " & FirstError.Message,
        NotificationType.Error
    )
)
```

Use this carefully. `FirstError.Message` may be too technical for end users.

---

## Safer User + Support Pattern

```powerfx
Set(
    varSupportReference,
    "INC-" & Text(Now(), "yyyymmddhhmmss")
);

IfError(
    Patch(
        Incidents,
        Defaults(Incidents),
        {
            Title: txtTitle.Text,
            Description: txtDescription.Text,
            SupportReference: varSupportReference
        }
    ),
    Notify(
        "We couldn't save your incident. Try again, or contact support and quote " & varSupportReference & ".",
        NotificationType.Error
    )
)
```

---

## Capturing Data Source Errors with `Errors`

After a failed data operation, use `Errors(DataSource)` to inspect data source errors.

```powerfx
If(
    !IsEmpty(Errors(Incidents)),
    Notify(
        First(Errors(Incidents)).Message,
        NotificationType.Error
    )
)
```

Example logging collection:

```powerfx
ClearCollect(
    colLastErrors,
    Errors(Incidents)
)
```

---

## Global App Error Handling with `App.OnError`

Use `App.OnError` for unhandled errors.

Example:

```powerfx
Notify(
    "Something unexpected happened. Please try again or contact support.",
    NotificationType.Error
);

Trace(
    FirstError.Message,
    TraceSeverity.Error,
    {
        Screen: App.ActiveScreen.Name,
        UserEmail: User().Email
    }
)
```

---

# Power Apps User-Friendly Error Messages

## Bad

```text
Error.
```

```text
Something went wrong.
```

```text
Patch failed: 403 connector exception.
```

## Better

```text
We couldn't save your incident. Please check your connection and try again.
```

```text
You do not have permission to update this record. Contact support if this looks wrong.
```

```text
The incident was saved, but the confirmation email could not be sent. You can continue working.
```

## Best Pattern

A good error message should explain:

1. what happened
2. whether the user needs to do anything
3. what they can do next
4. a support reference if needed

Example:

```text
We couldn't submit your incident because the approval service did not respond. Please try again. If it keeps happening, contact support and quote INC-1042.
```

---

# Power Automate Try/Catch/Finally Pattern

A production-ready cloud flow often uses three scopes:

```text
Trigger
  ↓
Try Scope
  ↓
Catch Scope
  ↓
Finally Scope
  ↓
Respond to Power Apps
```

## Try Scope

Put the main business logic here:

- create Dataverse row
- call external API
- send email
- update status
- perform approval logic

## Catch Scope

Configure **Run after** on the Catch Scope to run when Try has:

- failed
- timed out
- skipped

Use Catch for:

- logging failure
- creating support record
- notifying support
- returning a controlled error response

## Finally Scope

Configure Finally to run after both Try and Catch.

Use Finally for:

- cleanup
- final status update
- audit logging
- returning a final response

---

# Power Automate Flow Expressions

## Current Flow Run Name / ID

```text
workflow()?['run']?['name']
```

This is useful as a correlation ID or support reference.

Example support reference:

```text
concat('FLOW-', workflow()?['run']?['name'])
```

---

## Flow Name

```text
workflow()?['name']
```

---

## Trigger Name

```text
workflow()?['trigger']?['name']
```

---

## Full Workflow Object

```text
workflow()
```

Useful for debugging, but do not send the full object to users.

---

## Trigger Body

```text
triggerBody()
```

---

## Trigger Outputs

```text
triggerOutputs()
```

---

## Trigger Header

```text
triggerOutputs()?['headers']
```

Example, get a specific header:

```text
triggerOutputs()?['headers']?['x-ms-user-email']
```

---

## Action Body

```text
body('Action_Name')
```

Example:

```text
body('Get_a_row_by_ID')
```

---

## Action Outputs

```text
outputs('Action_Name')
```

---

## Action Status Code

Common for HTTP actions:

```text
outputs('HTTP')?['statusCode']
```

---

## Action Error Object

```text
outputs('Action_Name')?['error']
```

---

## Action Error Message

```text
outputs('Action_Name')?['error']?['message']
```

---

## Action Error Code

```text
outputs('Action_Name')?['error']?['code']
```

---

## Safe Fallback with `coalesce`

```text
coalesce(
    outputs('Action_Name')?['error']?['message'],
    'No error message was returned.'
)
```

---

## Convert Object to String

```text
string(outputs('Action_Name'))
```

Useful when logging raw output.

---

## Convert Result to JSON String for Logging

```text
string(result('Try'))
```

---

## Create a Support Reference

```text
concat(
    'ERR-',
    formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'),
    '-',
    substring(workflow()?['run']?['name'], 0, 8)
)
```

---

## Current UTC Time

```text
utcNow()
```

---

## Format Timestamp

```text
formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm:ss')
```

---

# Getting Flow Run Details

## Recommended Fields to Log

| Field | Expression |
|---|---|
| Flow name | `workflow()?['name']` |
| Flow run ID/name | `workflow()?['run']?['name']` |
| Trigger name | `workflow()?['trigger']?['name']` |
| Timestamp | `utcNow()` |
| Trigger payload | `string(triggerBody())` |
| Try scope result | `string(result('Try'))` |
| Failed actions | `string(filter(result('Try'), equals(item()?['status'], 'Failed')))` |
| Timed out actions | `string(filter(result('Try'), equals(item()?['status'], 'TimedOut')))` |

---

## Example Log Object

Use this in a Compose action or when creating a Dataverse/SharePoint log row.

```json
{
  "supportReference": "@{variables('varSupportReference')}",
  "flowName": "@{workflow()?['name']}",
  "flowRunId": "@{workflow()?['run']?['name']}",
  "timestamp": "@{utcNow()}",
  "tryResult": "@{string(result('Try'))}"
}
```

---

# Getting Error Details from Failed Actions

## Direct Failed Action Error Message

```text
outputs('Send_email')?['error']?['message']
```

## Direct Failed Action Error Code

```text
outputs('Send_email')?['error']?['code']
```

## HTTP Status Code

```text
outputs('HTTP')?['statusCode']
```

## HTTP Body Error Message

This depends on the API shape, but common examples include:

```text
body('HTTP')?['error']?['message']
```

```text
body('HTTP')?['message']
```

```text
body('HTTP')?['error_description']
```

Use `coalesce` when the shape can vary:

```text
coalesce(
    body('HTTP')?['error']?['message'],
    body('HTTP')?['message'],
    body('HTTP')?['error_description'],
    'The external service returned an error, but no message was provided.'
)
```

---

# Filtering Failed Results from a Try Scope

Power Automate/Logic Apps supports `result('ScopeName')`.

This returns an array of action results from inside the scope.

## Get All Failed Actions in Try Scope

```text
filter(
    result('Try'),
    equals(item()?['status'], 'Failed')
)
```

## Get All Timed Out Actions in Try Scope

```text
filter(
    result('Try'),
    equals(item()?['status'], 'TimedOut')
)
```

## Get Failed or Timed Out Actions

```text
filter(
    result('Try'),
    or(
        equals(item()?['status'], 'Failed'),
        equals(item()?['status'], 'TimedOut')
    )
)
```

## Get First Failed Action

```text
first(
    filter(
        result('Try'),
        equals(item()?['status'], 'Failed')
    )
)
```

## Get First Failed Action Name

```text
first(
    filter(
        result('Try'),
        equals(item()?['status'], 'Failed')
    )
)?['name']
```

## Get First Failed Action Error Code

```text
first(
    filter(
        result('Try'),
        equals(item()?['status'], 'Failed')
    )
)?['error']?['code']
```

## Get First Failed Action Error Message

```text
first(
    filter(
        result('Try'),
        equals(item()?['status'], 'Failed')
    )
)?['error']?['message']
```

## Safe First Failed Error Message

This avoids a blank/null issue if the failed action does not return an error message.

```text
coalesce(
    first(
        filter(
            result('Try'),
            equals(item()?['status'], 'Failed')
        )
    )?['error']?['message'],
    'The flow failed, but no detailed error message was returned.'
)
```

## Filter Failed, Timed Out or Cancelled Actions

```text
filter(
    result('Try'),
    or(
        equals(item()?['status'], 'Failed'),
        equals(item()?['status'], 'TimedOut'),
        equals(item()?['status'], 'Cancelled')
    )
)
```

## Count Failed Actions

```text
length(
    filter(
        result('Try'),
        equals(item()?['status'], 'Failed')
    )
)
```

## Check If Try Scope Had Failures

```text
greater(
    length(
        filter(
            result('Try'),
            equals(item()?['status'], 'Failed')
        )
    ),
    0
)
```

---

# Returning Errors Back to Power Apps

Use **Respond to a PowerApp or flow**.

Return a controlled object such as:

```json
{
  "success": false,
  "message": "We couldn't submit your incident. Please try again.",
  "supportReference": "ERR-20260520-ABC123"
}
```

## Example Response Values

| Field | Example |
|---|---|
| success | `false` |
| message | `We couldn't submit your incident. Please try again.` |
| supportReference | `variables('varSupportReference')` |
| flowRunId | `workflow()?['run']?['name']` |
| failedAction | `first(filter(result('Try'), equals(item()?['status'], 'Failed')))?['name']` |

---

## Power Apps Handling Flow Response

```powerfx
Set(
    varFlowResponse,
    SubmitIncidentFlow.Run(
        txtTitle.Text,
        txtDescription.Text
    )
);

If(
    varFlowResponse.success,
    Notify(
        "Incident submitted successfully.",
        NotificationType.Success
    ),
    Notify(
        varFlowResponse.message & " Quote " & varFlowResponse.supportReference,
        NotificationType.Error
    )
)
```

---

# Retry and Resilience Patterns

## When Retries Help

Retries are useful for transient failures such as:

- 408 request timeout
- 429 throttling
- 5xx server errors
- temporary connector or API availability issues

## When Retries Can Be Dangerous

Retries can create problems when an action is not idempotent.

Examples:

- creating duplicate records
- sending duplicate emails
- charging a payment twice
- creating duplicate tickets
- submitting duplicate approvals

## Safer Retry Pattern

Before creating a record, check whether it already exists using a correlation key.

Example key:

```text
concat(
    triggerBody()?['incidentId'],
    '-',
    workflow()?['run']?['name']
)
```

Or use a business-generated idempotency key from Power Apps:

```powerfx
Set(
    varSubmitId,
    GUID()
)
```

Pass `varSubmitId` into the flow and store it on the created row.

---

# Recommended Logging Table

Create a Dataverse table or SharePoint list called something like:

```text
Solution Error Log
```

Suggested columns:

| Column | Purpose |
|---|---|
| Support Reference | User-safe reference |
| Flow Name | Which flow failed |
| Flow Run ID | Flow run identifier |
| App Screen | Where the user was |
| User Email | Who experienced it |
| Error Message | Safe technical message |
| Failed Action | Which action failed |
| Raw Error JSON | Full technical context |
| Severity | Info / Warning / Error / Critical |
| Created On | Timestamp |
| Resolved | Support flag |

---

# Production Checklist

## Power Apps

- [ ] Required fields are validated
- [ ] Invalid formats are blocked
- [ ] Submit is disabled until safe
- [ ] User messages are clear and actionable
- [ ] Risky operations use `IfError`
- [ ] `App.OnError` is configured
- [ ] Data source errors are captured with `Errors`
- [ ] Users receive a support reference where needed
- [ ] Technical detail is logged, not dumped on the user

## Power Automate

- [ ] Main logic is inside a Try Scope
- [ ] Catch Scope uses Configure Run After
- [ ] Finally Scope handles cleanup/status updates
- [ ] Retry policies are intentional
- [ ] Failed actions are logged
- [ ] Flow run ID is captured
- [ ] Support reference is generated
- [ ] Response is returned to Power Apps
- [ ] Duplicate submit/idempotency is considered
- [ ] Owners know where to monitor failures

## Error Messages

- [ ] Tell the user what happened
- [ ] Tell the user what to do next
- [ ] Avoid raw connector/API errors
- [ ] Include a support reference for investigation
- [ ] Keep technical detail in logs

---

# Recommended Demo Flow

Use this simple live demo story:

1. Submit invalid data
2. Show validation stopping it
3. Submit valid data
4. Simulate a flow failure
5. Show Catch Scope logging the issue
6. Return a friendly error to Power Apps
7. Show support reference and flow run ID
8. Fix/retry the process
9. Show successful completion

---

# Speaker Lines

Use these in your session:

> Validation is the cheapest form of resilience.

> Error messages are part of UX design.

> The user does not need the stack trace. Support does.

> A failed flow is acceptable. A silent failed flow is not.

> Build for the happy path. Design for the bad day.

---

# Useful Expression Snippets

## Failed actions from Try

```text
filter(result('Try'), equals(item()?['status'], 'Failed'))
```

## First failed action name

```text
first(filter(result('Try'), equals(item()?['status'], 'Failed')))?['name']
```

## First failed message

```text
first(filter(result('Try'), equals(item()?['status'], 'Failed')))?['error']?['message']
```

## Flow run ID

```text
workflow()?['run']?['name']
```

## Safe support reference

```text
concat('ERR-', formatDateTime(utcNow(), 'yyyyMMdd-HHmmss'), '-', substring(workflow()?['run']?['name'], 0, 8))
```

## Full Try scope result as text

```text
string(result('Try'))
```

## Has failed actions

```text
greater(length(filter(result('Try'), equals(item()?['status'], 'Failed'))), 0)
```

---

## Notes

Expression syntax can vary slightly depending on where it is used in Power Automate:

- in expression editor, use the expression directly
- inside dynamic content strings, wrap expressions with `@{ ... }`
- action names must match exactly, including underscores generated by Power Automate
- scope names such as `Try` must match the actual scope name in the flow

