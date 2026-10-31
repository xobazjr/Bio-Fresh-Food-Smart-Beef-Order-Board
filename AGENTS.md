# AGENTS.md --- BIOFRESH FOOD Smart Beef Order Board

> Project guidance for AI coding agents and developers building the
> replacement system for the BIOFRESH FOOD Smart Beef Order Board (BOB
> Labels Pro).
>
> **Document status:** Initial implementation brief / draft
> requirements. Validate all business rules marked `CONFIRM` with
> BIOFRESH FOOD before treating them as final.
>
> **Reference prototype:**
> https://gemini-code-orderboard-truck.vercel.app/

------------------------------------------------------------------------

## 1. Mission and Product Scope

Build a centralized web-based **Order, Production, Quality, and
Logistics Management System** for BIOFRESH FOOD.

The system must support the operational lifecycle:

1.  Import or receive Sales Orders (SO), initially from TRCLOUD.
2.  Review and confirm orders in Sales.
3.  Plan production dates and shifts.
4.  Record actual production quantities and weights.
5.  Perform Quality Control (QC).
6.  Review production summaries.
7.  Schedule deliveries and assign trucks/drivers.
8.  Export operational Excel sheets.
9.  Archive completed orders and retain searchable history.

The product is more than a dashboard. It is a shared operational system
used by Sales, Planning, Production, QC, Logistics, and Management. All
departments must work from consistent, traceable data.

### Core principles

-   Keep one authoritative record for each business entity.
-   Avoid re-entering the same SO and product information between
    departments.
-   Preserve source SO data and record subsequent changes as auditable
    events.
-   Separate workflow status from operational readiness flags and
    measurement data.
-   Support traceability from SO → production plan → production record →
    lot/QC → delivery.
-   Do not hardcode business-configurable values such as truck count,
    shifts, users, product lists, or thresholds.
-   Never invent TRCLOUD API behavior, database structure, or
    undocumented business rules.

------------------------------------------------------------------------

## 2. Important Implementation-Agent Rules

Before changing code:

1.  Inspect the existing repository, package manifests, framework,
    routes, components, database schema, tests, and deployment
    configuration.
2.  Reuse established project conventions and dependencies where
    practical. Do not replace the stack without a clear reason.
3.  Identify existing functionality before implementing a duplicate.
4.  If the repository already contains a partially implemented module,
    preserve working behavior and make incremental changes.
5.  Do not assume that a UI element in the reference prototype is backed
    by a working API. Treat prototype observations as interface/workflow
    evidence, not proof of backend behavior.
6.  If a requirement is ambiguous, implement a configurable or safely
    extensible design and document the assumption. Do not silently
    invent a business rule.
7.  Keep secrets, API keys, database credentials, and tokens in
    environment variables. Never commit them.
8.  Validate user input on both client and server. The server is
    authoritative.
9.  Add automated tests for business calculations, permissions, state
    transitions, imports, and integrations.
10. Do not mark an order as delivered, QC-passed, or archived merely
    because a page was opened or a form was viewed.
11. Use Asia/Bangkok as the business timezone unless the client confirms
    otherwise. Store timestamps consistently (prefer UTC in storage,
    convert for display).
12. Before completion, run the available lint, type-check, test, and
    build commands. Report commands that could not be run.

### Required implementation discipline

-   Use clear domain names: `salesOrder`, `productionPlan`,
    `productionRecord`, `qcRecord`, `delivery`, `truck`, `productLot`.
-   Keep business logic out of presentation components where possible.
-   Use database transactions for multi-record operations.
-   Make import and synchronization operations idempotent.
-   Add pagination and server-side filtering for growing lists.
-   Provide loading, empty, success, validation-error, and failure
    states for every data-driven page.
-   Use confirmation dialogs for destructive or irreversible actions.
-   Record actor and timestamp for important changes.
-   Avoid destructive schema changes without a migration and a clear
    data-preservation plan.

------------------------------------------------------------------------

## 3. Roles and Access Control

Implement authentication and role-based authorization (RBAC).

  -----------------------------------------------------------------------
  Role                                Main responsibilities
  ----------------------------------- -----------------------------------
  `ADMIN`                             Full system configuration and
                                      access

  `SALES`                             Import, review, edit permitted
                                      fields, and confirm SOs

  `PRODUCTION_PLANNER`                Plan production dates and shifts

  `PRODUCTION_STAFF`                  View assigned work and record
                                      actual production

  `QC_STAFF`                          Perform inspections and record QC
                                      results

  `LOGISTICS`                         Schedule deliveries, assign
                                      trucks/drivers, and record delivery
                                      details

  `MANAGEMENT`                        Read dashboards, summaries, and
                                      reports
  -----------------------------------------------------------------------

### Authorization requirements

-   Enforce permissions in backend/API handlers, not only by hiding
    frontend menus.
-   A Sales user must not edit QC results.
-   A QC user must not change Sales Order source details.
-   Production staff must not change delivery assignments unless
    explicitly granted permission.
-   Management is read-only by default.
-   Admin can manage users, roles, master data, and system settings.
-   Record denied or sensitive operations where appropriate.
-   Define field-level permissions for fields that can be edited by only
    one department.

------------------------------------------------------------------------

## 4. End-to-End Business Workflow

Use the following as the initial workflow. Confirm exact status names
and transition rules with the client.

``` text
TRCLOUD / SO Import
        |
        v
Sales review and validation
        |
        v
Sales confirms SO
        |
        v
Production Planning
(assign cutting date and shift)
        |
        v
Production
(record actual quantity, gross/tare/net weight)
        |
        v
Quality Control
(record lot, dates, temperature, checklist)
        |
        v
Production Summary / Ready for Delivery
        |
        v
Logistics
(assign delivery date, truck, driver, destination, INV)
        |
        v
Delivery confirmation
        |
        v
Archive
```

### Suggested workflow states

Use stable machine-readable values and map them to localized labels in
the UI.

-   `DRAFT`
-   `PENDING_SALES_CONFIRMATION`
-   `CONFIRMED`
-   `PENDING_PLANNING`
-   `PLANNED`
-   `IN_PRODUCTION`
-   `PRODUCTION_COMPLETED`
-   `PENDING_QC`
-   `QC_FAILED`
-   `QC_PASSED`
-   `READY_FOR_DELIVERY`
-   `SCHEDULED_FOR_DELIVERY`
-   `IN_TRANSIT`
-   `DELIVERED`
-   `CANCELLED`
-   `ARCHIVED`

These are a proposed baseline, not confirmed prototype status values.
Refine them after workflow validation.

### State-management rules

-   Keep workflow state separate from readiness fields such as product
    availability.
-   A readiness toggle must not implicitly advance the workflow.
-   A state transition must be performed by an explicit action and
    validated server-side.
-   Reject invalid transitions and explain why.
-   Store state history with previous state, new state, actor,
    timestamp, and optional reason.
-   Do not allow QC failure to progress to delivery-ready without an
    authorized reinspection/pass process.
-   Archive only after the agreed delivery-completion condition is met.

------------------------------------------------------------------------

## 5. Functional Modules

## 5.1 Sales Order Management

### Purpose

Import and manage customer Sales Orders (SO), then pass confirmed orders
to Production Planning.

### Required capabilities

-   Import SO data from TRCLOUD API when available.
-   Provide a controlled Excel import fallback if API integration is
    unavailable.
-   Download an import-template workbook.
-   Preview and validate imported rows before committing.
-   Detect duplicate source documents.
-   Search by SO number, customer, product, and date.
-   Filter by SO date, requested delivery date, customer, Sales person,
    and status.
-   View SO header and all line items.
-   Edit permitted Sales-owned fields.
-   Save as draft and explicitly confirm.
-   View import and edit history.

### SO header fields

  Field                     Description
  ------------------------- ------------------------------------------
  `soNumber`                Unique source SO number
  `sourceSystem`            e.g. TRCLOUD or manual import
  `sourceDocumentId`        Source system identifier, if provided
  `soDate`                  Source document date
  `customerId`              Customer reference
  `salesPersonId`           Responsible Sales user
  `requestedDeliveryDate`   Customer requested delivery date
  `deliveryLocation`        Delivery destination
  `netAmount`               Source document net amount, if available
  `salesRemark`             Sales notes
  `sourceDocumentUrl`       Link to source document, if available
  `workflowStatus`          Current workflow state

### SO line-item fields

-   Product code and product reference
-   Product name snapshot
-   Ordered quantity
-   Unit of measure
-   Ordered weight, where applicable
-   Unit price and line amount, if supplied by source
-   Product/specification notes
-   Source line identifier, if available

### Rules

-   One SO can contain multiple products.
-   Enforce uniqueness using the source system plus stable source
    document identifier or SO number.
-   Re-importing the same source SO must not create duplicates.
-   Preserve the original imported payload or a normalized source
    snapshot for audit/debugging.
-   Do not overwrite locally maintained fields when refreshing source
    data unless an explicit field ownership rule permits it.
-   Confirm whether confirmed SOs can be edited, cancelled, or amended.
    Record amendments as revisions/events rather than silently replacing
    history.

------------------------------------------------------------------------

## 5.2 Production Planning

### Purpose

Plan confirmed SO items for cutting/production by date and shift.

### Required capabilities

-   List confirmed orders awaiting planning.
-   Assign cutting/production date.
-   Assign production shift.
-   Add planning remarks.
-   Set product readiness separately from workflow status.
-   Edit plans with permission and audit history.
-   View daily and weekly planning boards.
-   View a calendar of planned work.
-   Filter by date, shift, SO, customer, Sales person, product, and
    status.
-   Show total planned order weight and item counts per day/shift.
-   Support planning ahead, including next-week planning.

### Planning fields

-   SO and SO line reference
-   Planned production/cutting date
-   Shift
-   Planned quantity/weight
-   Planning remark
-   Readiness flag/status
-   Planner and timestamps

### Confirm with client

-   Working days and holidays.
-   Number and names of shifts.
-   Whether one SO line can be split across multiple production dates or
    shifts.
-   Whether one plan can combine items from multiple SOs.
-   Whether a plan can be re-assigned after production starts.

------------------------------------------------------------------------

## 5.3 Production Management

### Purpose

Provide a production board and record actual output.

### Required capabilities

-   Daily production board.
-   Filter by production date, shift, SO, customer, product, and status.
-   Open SO and product-line details.
-   Record actual produced quantity and weight.
-   Record package/piece counts where relevant.
-   Record gross weight, tare weight, and net weight separately.
-   Record tare status and measurement details.
-   Record completion date/time and production remarks.
-   Show ordered vs actual values and variance.
-   Preserve edits and measurement history.
-   Support authorized corrections without erasing previous values.

### Suggested production record fields

-   `productionPlanId`
-   `salesOrderItemId`
-   `productId`
-   `producedQuantity`
-   `packageCount`
-   `grossWeight`
-   `tareWeight`
-   `netWeight`
-   `tareApplied`
-   `measuredAt`
-   `measuredBy`
-   `completedAt`
-   `productionRemark`

### Weight calculations

Use decimal-safe arithmetic. Do not use floating-point arithmetic for
persisted commercial weights.

``` text
netWeight = grossWeight - tareWeight
totalActualNetWeight = sum(netWeight for applicable production records)
variance = totalActualNetWeight - orderedWeight
```

Validation:

-   Gross weight and tare weight cannot be negative.
-   Tare weight cannot exceed gross weight.
-   Net weight cannot be negative.
-   Unit conversions must be explicit and consistent.
-   Clearly distinguish quantity (pieces/packages) from weight (kg).
-   Do not assume every product is sold or planned by kg; use
    product-specific units.
-   Show whether the recorded weight is gross, tare, or net.

### Example

``` text
Product A: ordered 100 kg, actual net 98 kg
Product B: ordered  70 kg, actual net 72 kg
Total ordered: 170 kg
Total actual:  170 kg
Variance:        0 kg
```

### TRCLOUD outbound data

If supported by the official TRCLOUD API and confirmed by the client,
consider syncing:

-   SO number / source document identifier
-   Product code
-   Actual quantity
-   Actual net weight
-   Production date
-   Relevant production remark

Do not implement assumed TRCLOUD endpoints. First inspect official API
documentation and credentials supplied by the client.

------------------------------------------------------------------------

## 5.4 Quality Control (QC)

### Purpose

Record quality inspections before products are released for delivery.

### Suggested QC fields

-   SO / SO line / production record reference
-   Product and lot number
-   MFG date
-   EXP date
-   Product temperature
-   Preparation time
-   Labeling result
-   Foreign-matter inspection result
-   Vacuum/packaging result
-   QC remark
-   Inspector
-   Inspection timestamp
-   Overall result

### Initial checklist

-   Label is correct and complete.
-   No foreign matter is detected.
-   Vacuum and packaging are acceptable.
-   MFG and EXP information is valid.
-   Lot number is recorded.
-   Product temperature meets the approved threshold.

### Temperature rule

The prototype was reported to show a chilled-meat threshold of **-1.5°C
or colder**. Treat this as **unconfirmed** until BIOFRESH FOOD QC
management approves the specification.

If approved, for a maximum permitted temperature of `-1.5°C`, pass when:

``` text
measuredTemperature <= -1.5
```

Store the threshold as configurable QC master data, including unit,
effective date, product/category scope, and approver. Do not hardcode it
throughout the application.

### QC outcomes

-   `PENDING`
-   `PASSED`
-   `FAILED`
-   `REINSPECTION_REQUIRED`

A failed inspection must record failed checks and a reason. Reinspection
must create a new inspection record or revision; do not erase the failed
inspection.

Confirm whether one SO line can contain multiple lots and whether QC is
performed per item, package, lot, or order.

------------------------------------------------------------------------

## 5.5 Production Summary

### Purpose

Provide a consolidated review of order, production, and QC information
before delivery planning.

Display:

-   Ordered quantity and weight
-   Actual quantity and weight
-   Gross, tare, and net weight where relevant
-   Variance
-   Tare status
-   Actual production completion time
-   Lot number(s)
-   MFG / EXP dates
-   QC status and inspection details
-   Planned and actual delivery dates

Viewing the summary must not change workflow state. Editing data must
follow role permissions and be audited.

------------------------------------------------------------------------

## 5.6 Transportation and Cold-Chain Logistics

### Purpose

Schedule completed, QC-released orders for delivery and manage truck
capacity.

### Delivery fields

-   Delivery/order reference
-   SO reference
-   Customer
-   Delivery destination
-   Planned delivery date/time
-   Actual delivery date/time
-   Truck
-   Driver
-   Receiver/contact, where appropriate
-   Invoice (`INV`) number
-   Delivery weight
-   Delivery status
-   Logistics remark
-   Proof-of-delivery attachment/reference, if required

### Truck master data

-   Truck ID
-   Display name
-   License plate
-   Maximum load capacity
-   Tare weight, if operationally relevant
-   Active/inactive status
-   Optional temperature-control capability/details
-   Assigned/default driver, if applicable

The prototype was described as showing 10 trucks. Do not hardcode a
10-truck limit. Truck count and capacity must be configurable master
data.

### Scheduling capabilities

-   Assign one or more SOs to a truck and delivery run.
-   View daily and 7-day truck schedules.
-   Filter by delivery date, truck, driver, customer, and status.
-   Show all orders assigned to a truck.
-   Edit assignments with permission and audit history.
-   Prevent accidental double-booking if the business requires exclusive
    truck/time allocation.
-   Support multiple orders per truck.
-   Design the data model to allow one SO to be split across multiple
    deliveries if confirmed by the client.

### Capacity validation

Calculate the load using the weight basis approved by the client (gross
or net; do not assume).

``` text
truckLoad = sum(delivery load weights assigned to the truck/run)
remainingCapacity = maximumLoadCapacity - truckLoad
```

If `truckLoad > maximumLoadCapacity`, show an overload warning and block
confirmation unless an explicitly authorized override policy is
approved. Record any override and reason.

Example:

``` text
Maximum capacity: 1,200 kg
Current load:     1,350 kg
Over capacity:     150 kg
Status: OVERLOAD
```

### Calendar statuses

Suggested display statuses:

-   Available
-   Assigned
-   In delivery

These are visual scheduling statuses; keep them distinct from each SO's
workflow state.

### Confirm with client

-   Capacity calculation uses gross weight or net weight.
-   Whether tare/packaging weight is included.
-   Whether one truck can have multiple delivery runs per day.
-   Whether one run can include multiple destinations.
-   Whether one SO can be split between trucks or delivery dates.
-   Driver assignment and vehicle availability rules.
-   Required temperature records during transport.

------------------------------------------------------------------------

## 5.7 Butcher Production Sheet

### Purpose

Aggregate planned production demand for a selected day to help the
cutting/production team prepare work.

### Required capabilities

-   Select production date.
-   Aggregate all applicable planned SO items for that date.
-   Group totals by product and specification.
-   Show each SO/customer contribution for traceability.
-   Display order count and total planned weight.
-   Export a printable/downloadable production sheet.

Example:

``` text
Production date: 22/10/2026

SO001 / Customer A
  Product A: 50 kg
  Product B: 20 kg

SO002 / Customer B
  Product A: 30 kg
  Product C: 40 kg

Daily totals
  Product A: 80 kg
  Product B: 20 kg
  Product C: 40 kg
  Total:    140 kg
```

Do not aggregate away the source SO references. Users must be able to
drill down from a product total to the contributing order lines.

------------------------------------------------------------------------

## 5.8 Excel Import and Export

Provide validated Excel workflows with documented templates.

### Import template

-   Provide a downloadable sample/template for SO import when required.
-   Validate required columns, data types, dates, product codes, units,
    and duplicate rows.
-   Show row-level errors before committing.
-   Provide a downloadable error report.
-   Do not partially import a workbook unless the user explicitly
    chooses a supported partial-import mode.

### Daily Production Plan export

-   Select production date.
-   Include all applicable shifts if requested by the business.
-   One product/order line per row.
-   Include SO, customer, product, planned quantity/weight,
    specification, and delivery deadline.
-   Leave actual production-entry columns blank if the sheet is intended
    for factory completion.
-   Keep unspecified product specifications blank rather than inventing
    values.

### Truck Schedule export

-   Select delivery date.
-   Include SO, customer, destination, truck, driver, product/quantity,
    and relevant delivery information.
-   Group or sort by truck.
-   Distinguish ordered quantity from actual weight.
-   Confirm whether the export includes only scheduled/archived orders
    or all delivery records for the selected date.

### General export rules

-   Export must respect current filters and permissions.
-   Use clear date/time and unit formatting.
-   Prevent formula injection from user-controlled cell values.
-   Include export generation time and selected filters where
    appropriate.
-   Test exported workbooks with realistic multi-order data.

------------------------------------------------------------------------

## 5.9 Archive

### Purpose

Retain completed order and delivery records for historical lookup and
audit.

### Required capabilities

-   Archive an order only when the approved completion criteria are met.
-   Search by SO number, customer, product, date, truck, driver, and
    invoice.
-   View complete order, production, QC, lot, and delivery history.
-   Preserve links to original source data and attachments.
-   Export historical records when authorized.

Archive should be a lifecycle state or queryable view over retained
records, not a destructive move that breaks foreign-key relationships or
removes operational history.

Confirm whether archive eligibility requires delivery confirmation,
invoice completion, receiver confirmation, or another condition.

------------------------------------------------------------------------

## 5.10 Dashboard and Reports

### Dashboard metrics

-   Total orders
-   Orders awaiting planning
-   Planned orders
-   Orders in production
-   Orders awaiting QC
-   QC failures
-   Ready-for-delivery orders
-   Delivered orders
-   Truck utilization
-   Truck overload alerts

Metrics must have clear definitions, date ranges, and timezone behavior.
Dashboard totals must reconcile with the underlying filtered records.

### Suggested reports

-   Daily/weekly SO volume
-   Planned vs actual production
-   Weight variance
-   QC pass/fail and failure reasons
-   Lot traceability
-   Delivery performance
-   Truck utilization and overloads
-   Department workflow aging

------------------------------------------------------------------------

## 6. Notifications

Notifications are recommended enhancements; confirm channels and timing
with the client.

Potential events:

-   New SO imported
-   SO awaiting Sales confirmation
-   SO awaiting production planning
-   Production due/overdue
-   Production completed
-   QC failed or reinspection required
-   Order ready for delivery
-   Truck capacity exceeded
-   Delivery schedule changed
-   Delivery completed
-   TRCLOUD synchronization failed

Support in-app notifications first. LINE or other external channels
should be implemented only after the client confirms the provider,
recipients, templates, and permissions.

------------------------------------------------------------------------

## 7. Audit Trail and Traceability

Auditability is a core requirement.

For important changes, record:

-   Entity type and entity ID
-   Action/event
-   Previous value and new value, where applicable
-   Actor/user ID
-   Timestamp
-   Reason/comment, where required
-   Request/correlation ID for API operations

At minimum, audit:

-   SO import, edit, confirmation, cancellation
-   Production plan creation and changes
-   Production measurements and corrections
-   QC inspections and reinspection
-   Lot assignments
-   Delivery/truck/driver assignments
-   Capacity overrides
-   Workflow state transitions
-   TRCLOUD import/export attempts
-   Archive action

Do not silently overwrite historical measurements or inspection results.
Preserve revisions/events.

Traceability should allow users to navigate:

``` text
Customer
  → Sales Order
    → SO line
      → Production plan
        → Production record
          → Product lot
            → QC inspection
              → Delivery
                → Truck / Driver / Invoice
```

------------------------------------------------------------------------

## 8. Suggested Data Model

This is a starting domain model, not a claim about the prototype's
actual database.

  Entity/table              Purpose
  ------------------------- --------------------------------------------
  `users`                   User accounts
  `roles`                   Role definitions
  `permissions`             Permission definitions
  `user_roles`              User-role assignments
  `customers`               Customer master
  `products`                Product master
  `product_units`           Units and conversions
  `sales_orders`            SO header
  `sales_order_items`       SO line items
  `sales_order_revisions`   SO amendment history, if needed
  `production_shifts`       Shift master
  `production_plans`        Planned production work
  `production_plan_items`   Planned quantities by SO line/date/shift
  `production_records`      Actual production records
  `product_lots`            Lot/batch records
  `qc_records`              Inspection header/results
  `qc_check_items`          QC checklist results
  `trucks`                  Truck master
  `drivers`                 Driver master
  `delivery_runs`           A truck's scheduled run
  `deliveries`              Delivery records
  `delivery_items`          SO lines/quantities assigned to a delivery
  `invoices`                Invoice references
  `order_status_history`    Workflow transitions
  `activity_logs`           Audit events
  `integration_logs`        External API/import logs
  `attachments`             Documents and proof files
  `notification_events`     Notification history

### Data-model design notes

-   Use primary keys independent of human-readable SO numbers.
-   Add unique constraints for external source identifiers.
-   Use foreign keys and appropriate indexes.
-   Use decimal columns for weights, quantities, and money where
    applicable.
-   Store units explicitly; do not assume all quantities are kg.
-   Model many-to-many and split workflows explicitly (SO lines to
    production plans, SO lines to deliveries).
-   Consider immutable source snapshots for imported SO data.
-   Use soft deletion only where appropriate; never use it as a
    substitute for audit history.
-   Add created/updated timestamps and actor IDs to operational
    entities.
-   Use migrations and seed data for configurable master data.

------------------------------------------------------------------------

## 9. API Design Starting Point

These endpoint names are illustrative. Adapt them to the existing
codebase and API conventions.

  Method    Endpoint                             Purpose
  --------- ------------------------------------ -----------------------------------
  `GET`     `/api/sales-orders`                  Search/list SOs
  `GET`     `/api/sales-orders/:id`              SO details
  `POST`    `/api/sales-orders/import`           Import SO workbook/data
  `POST`    `/api/sales-orders/:id/confirm`      Confirm SO
  `PATCH`   `/api/sales-orders/:id`              Update permitted SO fields
  `GET`     `/api/production-plans`              List/filter plans
  `POST`    `/api/production-plans`              Create plan
  `PATCH`   `/api/production-plans/:id`          Update plan
  `POST`    `/api/production-records`            Record production
  `POST`    `/api/qc-records`                    Create QC inspection
  `POST`    `/api/qc-records/:id/reinspect`      Record reinspection
  `GET`     `/api/trucks`                        List trucks
  `POST`    `/api/delivery-runs`                 Create truck run
  `PATCH`   `/api/delivery-runs/:id`             Update truck run
  `POST`    `/api/deliveries`                    Create delivery
  `GET`     `/api/archives`                      Search archived/completed records
  `GET`     `/api/reports/production`            Production report
  `GET`     `/api/reports/delivery`              Delivery report
  `GET`     `/api/integrations/trcloud/status`   Integration status

### API requirements

-   Validate authentication and permissions on every protected route.
-   Validate request bodies with shared schemas.
-   Return consistent error structures and suitable HTTP status codes.
-   Paginate collection endpoints.
-   Support explicit sorting and filtering.
-   Use idempotency keys or source identifiers for import/sync
    operations.
-   Use transactions for multi-entity writes.
-   Avoid exposing internal stack traces or secrets.
-   Document endpoints and payloads (OpenAPI is recommended).

------------------------------------------------------------------------

## 10. TRCLOUD Integration

Treat TRCLOUD as an external system whose exact capabilities must be
verified.

### Inbound (TRCLOUD → BOB)

Potential data:

-   SO identifier and number
-   SO date
-   Customer
-   Product code/name
-   Ordered quantity and unit
-   Ordered weight, if supplied
-   Amounts, if required
-   Document link

### Outbound (BOB → TRCLOUD)

Potential data, subject to official API support and client approval:

-   SO reference
-   Product code
-   Actual quantity
-   Actual net weight
-   Production date
-   Production remark

### Integration safeguards

-   Obtain official API documentation and authorized credentials.
-   Never assume endpoint paths, authentication method, rate limits, or
    payload formats.
-   Keep credentials in server-side environment variables/secrets.
-   Log request metadata, response status, and sanitized error details.
-   Do not log secrets or unnecessary personal/customer data.
-   Implement retries with backoff for transient errors.
-   Make synchronization idempotent.
-   Track sync status, last attempt, last success, and error details.
-   Provide manual retry for authorized users.
-   Do not mark local work complete solely because a sync was attempted.
-   Provide Excel import/export fallback if the API is unavailable or
    not included in the client's TRCLOUD plan.

------------------------------------------------------------------------

## 11. Non-Functional Requirements

  Area              Requirement
  ----------------- --------------------------------------------------------
  Responsive UI     Support desktop, tablet, and mobile workflows
  Performance       Paginate and filter large lists server-side
  Security          Authentication, RBAC, secure secrets, input validation
  Reliability       Transactional writes and recoverable integration jobs
  Data integrity    Foreign keys, constraints, idempotent imports
  Auditability      Track important actions and changes
  Backup            Define automated database backup and restore procedure
  Timezone          Display operational times in `Asia/Bangkok`
  Accessibility     Labels, keyboard navigation, readable contrast
  Maintainability   Modular domain logic and consistent conventions
  Scalability       Configurable products, shifts, trucks, and users
  Export            Safe, reproducible Excel exports
  Observability     Structured logs and actionable error reporting

------------------------------------------------------------------------

## 12. UI / Page Inventory

Build only what is needed for the agreed phase, but keep navigation and
domain boundaries compatible with this inventory.

1.  Login
2.  Dashboard
3.  Sales Order list
4.  Sales Order detail
5.  SO import/preview
6.  Production planning board
7.  Production calendar
8.  Production board
9.  Production detail / actual output entry
10. QC inspection
11. Production summary
12. Delivery management
13. Truck schedule (daily/7-day)
14. Truck detail
15. Butcher production sheet
16. Archive/history
17. Reports
18. User and role management
19. Master data management
20. TRCLOUD integration settings/status

Every data page should provide appropriate loading, empty, error,
success, validation, and permission-denied states.

------------------------------------------------------------------------

## 13. Delivery Plan and Priorities

### Phase 1 --- Core Operations (Critical)

-   Authentication and roles
-   Customer/product master data (minimum required)
-   SO import and Sales review/confirmation
-   Production planning
-   Production board
-   Actual production and weight recording
-   Basic dashboard
-   Basic audit history

**Phase 1 outcome:** An order can be imported, confirmed, planned,
produced, and measured.

### Phase 2 --- Quality and Delivery (High)

-   QC checklist and lot records
-   Production summary
-   Truck and driver master data
-   Delivery scheduling and capacity validation
-   Truck calendar
-   Excel exports
-   Delivery completion

**Phase 2 outcome:** An order can pass QC, be assigned to transport, and
be completed.

### Phase 3 --- Optimization (Medium)

-   Archive and advanced search
-   Management reports
-   Notifications
-   Advanced audit views
-   Advanced TRCLOUD synchronization
-   Additional traceability and operational analytics

Prioritize confirmed business needs over speculative enhancements.

------------------------------------------------------------------------

## 14. Acceptance Criteria

The following are minimum end-to-end acceptance scenarios. Add automated
tests where possible.

### SO import

-   Given a valid SO file/source payload, the system previews and
    validates it.
-   A valid SO can be imported without duplicate creation.
-   An invalid row shows a useful row-level error.
-   Re-importing the same source SO is idempotent.
-   Imported source data can be traced.

### Sales confirmation

-   A Sales user can review and confirm an eligible SO.
-   An unauthorized user cannot confirm it.
-   Confirmation is recorded in status history.
-   A confirmed SO becomes available to Planning.

### Production planning

-   A planner can assign a date and shift to eligible SO lines.
-   Daily/weekly boards show the assigned work.
-   Totals reconcile with included plan items.
-   Plan changes are audited.
-   Readiness changes do not silently advance workflow state.

### Production recording

-   Production staff can record quantity and weight for authorized work.
-   Gross, tare, and net values are validated.
-   Net weight and variance calculations are correct.
-   Corrections preserve previous values in history.
-   Totals reconcile across production records.

### QC

-   An inspector can record each required check and the result.
-   A failed inspection records its reason.
-   Failed QC cannot be treated as passed without an authorized
    reinspection/pass record.
-   Approved temperature thresholds are configurable and tested.
-   Lot and inspection history can be retrieved.

### Delivery

-   Logistics can schedule eligible QC-released orders.
-   Truck load totals are calculated using the approved weight basis.
-   Over-capacity assignments are blocked or handled by an explicitly
    approved override.
-   Multiple orders per truck/run are supported if required.
-   Delivery completion is recorded with actor and timestamp.

### Archive and reporting

-   Only eligible completed records can be archived.
-   Archived records retain their production, QC, lot, and delivery
    history.
-   Search and report totals reconcile with source records.

### Security

-   Backend authorization blocks forbidden actions even if a user calls
    the API directly.
-   Sensitive configuration is not exposed to the browser.
-   Audit records identify who changed critical data.

------------------------------------------------------------------------

## 15. Open Questions --- Client Confirmation Required

Do not finalize business logic until these are answered.

### TRCLOUD

1.  Is API access available on the client's current plan?
2.  Which SO fields can be retrieved?
3.  Which production/actual-weight fields can be written back?
4.  Is Excel import/export required as a permanent fallback?

### Sales Orders

5.  Can a confirmed SO be edited or cancelled?
6.  Can one SO be delivered on multiple dates?
7.  Can the same SO line be split across production dates?

### Production

8.  What are the working days, holidays, and shifts?
9.  Can one plan combine several SOs?
10. What are the exact product units and weight rules?
11. How is tare measured and applied?
12. Is a scale connected directly to the application? If yes, what
    model/protocol/interface?

### QC

13. Is `-1.5°C` the approved threshold, and does it apply to all chilled
    products?
14. What is the complete QC checklist?
15. Is QC performed per product, package, lot, or order?
16. Can an order/product have multiple lots?
17. What is the reinspection and rejection workflow?

### Logistics

18. What are the actual truck capacities?
19. Is capacity based on gross or net weight?
20. Can one truck make multiple runs per day?
21. Can one run serve multiple destinations?
22. Can one SO be split across trucks?
23. Is transport temperature monitoring required?
24. What constitutes delivery completion?

### Operations and users

25. How many users and users per department are expected?
26. Are approvals required at any stage?
27. Which notification channels are required?
28. What are the exact Excel templates and column orders?
29. What is the retention policy for orders, QC, and audit records?
30. Which reports are mandatory for the first release?

------------------------------------------------------------------------

## 16. Technical Decisions to Make After Repository Inspection

Do not assume a technology stack from this document. Inspect the
repository first, then document:

-   Frontend framework and routing
-   Backend framework and API conventions
-   Database and migration tooling
-   Authentication provider/session strategy
-   File storage for Excel and attachments
-   Background job/queue strategy for imports and integrations
-   Realtime update strategy, if required
-   Deployment environment and environment variables
-   Logging, monitoring, backup, and restore process

Prefer the smallest architecture that safely supports the required
workflow. Avoid introducing microservices unless scale or integration
boundaries justify them.

------------------------------------------------------------------------

## 17. Definition of Done

A feature is complete only when:

-   The user flow is implemented end-to-end.
-   Backend validation and authorization are implemented.
-   Database migrations are included where needed.
-   Loading, empty, error, and success UI states exist.
-   Business rules have automated tests.
-   Audit history is recorded for critical changes.
-   No secrets are committed.
-   Relevant documentation is updated.
-   Lint, type-check, tests, and production build pass (or blockers are
    clearly reported).
-   Any unresolved assumption is documented and not presented as
    confirmed business behavior.

------------------------------------------------------------------------

## 18. Final Guidance

Build a dependable operational system, not just a visual copy of the
prototype.

The most important engineering risks are:

1.  Incorrect relationships between SO lines, production plans, lots,
    and split deliveries.
2.  Inconsistent weight/unit calculations.
3.  Workflow state being changed implicitly by unrelated fields.
4.  Duplicate or conflicting TRCLOUD imports/synchronizations.
5.  Missing audit history for production measurements and QC.
6.  Truck capacity being calculated with the wrong weight basis.
7.  Hardcoded operational settings that should be configurable.

Resolve these risks in the domain model and backend before polishing the
UI. When in doubt, preserve data, make the rule explicit, and ask for
client confirmation rather than guessing.
