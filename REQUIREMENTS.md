# Group 9 — CrisisSync Requirements

## Team Members

| Name | ID Number | College Email |
|---|---|---|
| Mayar Serhan | 325475440 | Mayar.Serhan@e.braude.ac.il |
| Firas Ghaziani | 325486645 | Firas.ghaziani@e.braude.ac.il |
| Ahmad Tkatka | 212790125 | Ahmad.Tkatka@e.braude.ac.il |
| Osama Asdi | 324238559 | Osama.Asdi@e.braude.ac.il |
| Nadeer Faris | 327636817 | Nadeer.faris@e.braude.ac.il |

## Functional Requirements

FR-01: The system shall enforce role-based access control (RBAC) server-side across all reservation, task-completion, and SLA reporting operations, restricting manual and automatic resource reservation to `Dispatcher`, task completion reporting to the appointed `Field Commander`, and on-demand SLA report generation and viewing to `System Administrator`.
FR-02: The system shall treat each personnel and equipment resource as an atomic entity and ensure it is reserved for at most one task at any time.
FR-03: The system shall prevent personnel in `Unavailable` status and equipment in `Faulty/In Maintenance` status from being reserved for any task.
FR-04: The system shall treat a `Field Commander` as both an active system user and a personnel resource; when reserved for an `Active` task, the `Field Commander` shall continue operating normally in the system while being blocked from concurrent reservation to any other task.
FR-05: The system shall require every task resource package to contain exactly one `Field Commander`, rejecting any reservation request containing zero or more than one `Field Commander` with an explicit validation error.
FR-06: The system shall prevent resource reservation for any task in the `Blocked` state until all of its predecessor tasks reach `Completed` and the server transitions the task to `Ready`.
FR-07: The system shall allow an authorized `Dispatcher` to submit a manual resource-package reservation for a `Ready` task directly from the live incident status screen.
FR-08: Using the relational SQL database as the single source of truth, the system shall execute every resource-package reservation inside an ACID SQL transaction with pessimistic row locking on both the target task and all requested resource rows, performing authoritative verification of task state (`Ready`) and resource availability (`Available`) inside the transaction after acquiring the locks (optional pre-checks prior to locking may fail fast, without replacing the in-transaction authoritative check).
FR-09: Upon successful in-transaction verification, the system shall reserve the entire requested resource package atomically (updating all requested resources to `Reserved for Task`) and transition the task from `Ready` to `Active`.
FR-10: If any requested resource is unavailable, faulty, or already locked, if the target task is no longer in `Ready` state upon lock acquisition, or if lock-wait times out during manual reservation, the system shall roll back the failed transaction completely—committing no task or resource changes and without undoing changes committed by concurrent transactions—notify the `Dispatcher` with the conflicting resource identifier or failure reason, and update the `Dispatcher`'s screen to reflect current database state.
FR-11: The system shall allow an authorized `Dispatcher` to request automatic resource allocation for a `Ready` task according to a selectable allocation criterion (such as closest geographic proximity to the incident location).
FR-12: When concurrent automatic allocations compete for the same resource, the system shall grant the resource to only one allocation and automatically attempt up to two next-best available resource alternatives for the competing allocation without requiring `Dispatcher` resubmission and without blocking unrelated server requests. `[Team Assumption]` The automatic allocator caps contention retries at two fallback attempts (three total attempts: initial candidate plus up to two next-best alternatives) to bound server latency and prevent request starvation; if a fourth or further matching `Available` candidate exists in the pool after all three attempts fail under contention, the request aborts cleanly and instructs the `Dispatcher` to retry or reserve manually.
FR-13: Upon successful automatic allocation (initial selection or up to two fallback attempts), the system shall lock and reserve the selected package atomically (`Ready` to `Active`); if no valid alternative succeeds within the retry limit or none exists in the pool, the system shall abort with zero partial reservations (without undoing concurrent committed transactions) and notify the `Dispatcher`.
FR-14: The system shall allow only the `Field Commander` appointed to an `Active` task to report its completion from the incident status screen, rejecting any task-completion submission from an unappointed `Field Commander`, another user role, or for any task not in the `Active` state.
FR-15: Upon valid completion reporting of an `Active` task, the system shall transition the task to the immutable `Completed` state, immediately release the task reservation so resources still in `Reserved for Task` return to `Available` while any equipment already transitioned to `Faulty/In Maintenance` mid-task retains its `Faulty/In Maintenance` status, and automatically promote any dependent `Blocked` task to `Ready` once all of its predecessor tasks are `Completed`.
FR-16: The system shall push every task-state transition and resource-status change from the server immediately to all relevant connected client workstations via OCSF Server Push without client-side polling.
FR-17: The system shall automatically generate and persist the monthly SLA report for the preceding calendar month after month-end according to the synchronized server clock (`[Team Assumption]` triggered at `00:00:00 UTC` on the first day of the following month so final-millisecond events of the closing month are captured), and shall allow only the `System Administrator` to generate on-demand SLA reports for a selected past month and view generated reports.
FR-18: The system shall calculate the monthly SLA report per incident type (`Building Collapse`, `Fire`, `Traffic Accident`, `HazMat Leak`, and `Natural Disaster`) as the average elapsed time from incident opening to task closure across qualifying incidents. `[Team Assumption]` Task closure is defined as the latest final-state (`Completed` or `Cancelled`) transition timestamp among all tasks in the incident; an incident is attributed to the target calendar month (UTC) in which its last task reached a final state (excluding still-open incidents); and `Cancelled` incidents (e.g., false alarms) as well as incidents with zero `Completed` tasks are excluded so they do not skew SLA metrics.
FR-19: When a reporting period or incident type has zero qualifying completed incidents, the system shall output `N/A` (never `0`) without failing report generation.
FR-20: The system shall record every resource reservation, task-completion transition, and SLA report generation event in an immutable (append-only) audit log together with the exact server UTC timestamp and acting user identifier (`SYSTEM` for automated scheduled monthly SLA generation; authenticated user identifier for all user-initiated actions).

## Non-Functional Requirements

NFR-01: `[Team Target]` Under normal uncontested load (up to 50 concurrent active workstations), the system shall complete an uncontested manual or automatic resource-package reservation SQL transaction within ≤ 1.5 seconds, measured from server receipt of the OCSF reservation request to SQL transaction commit and response dispatch.
NFR-02: `[Team Target]` Under resource contention, database pessimistic row locking shall enforce a lock-wait timeout of ≤ 3.0 seconds; any conflicted, fallback-retried, or timed-out reservation request shall complete rollback/retry and dispatch its final server response within ≤ 3.5 seconds from server receipt, with each automatic fallback attempt adding ≤ 500 milliseconds and introducing ≤ 100 milliseconds of delay to unrelated active requests.
NFR-03: `[Team Target]` Across a measurement series of 100 consecutive task-completion submissions under normal load, the 95th-percentile server processing latency (state transition, resource release, and dependent-task promotion) shall be ≤ 1.0 second, measured from server request receipt to SQL commit.
NFR-04: `[Team Target]` Across a measurement series of 100 consecutive task-state or resource-status changes, the server shall broadcast OCSF Server Push updates to all relevant connected workstations with a 95th-percentile delivery latency of ≤ 1.0 second (measured from SQL commit to client receipt on LAN), 0 periodic client-side polling requests, and ≤ 64 KB average delta-push message size.
NFR-05: `[Team Target]` The live incident status screen shall load and render an incident with up to 200 tasks within ≤ 2.0 seconds while supporting at least 50 concurrently connected workstations; the SLA reporting engine shall compute and render a monthly report across up to 10,000 historical task records within ≤ 3.0 seconds (on-demand) and ≤ 10.0 seconds (scheduled batch execution).
NFR-06: `[Team Target]` The server shall enforce role-based access control (RBAC) server-side on 100% of incoming OCSF reservation, task-completion, and SLA-reporting requests and reject unauthorized operations within ≤ 100 milliseconds from request receipt.
NFR-07: `[Team Target]` All client-server communications over LAN TCP/IP (OCSF) shall be encrypted using TLS 1.2 or TLS 1.3 with AES-256 cipher suites; user passwords shall be stored as salted BCrypt hashes; and database credentials shall be stored encrypted using AES-256, with 0 plaintext credentials, internal stack traces, or SQL schemas exposed in application logs or client error dialogs.
NFR-08: `[Team Target]` All SQL/ORM queries used for reservations, task completion, audit logging, and SLA aggregation shall use 100% parameterized queries or prepared statements to prevent SQL Injection (0 vulnerabilities), and the audit log table shall enforce database privileges blocking 100% of `UPDATE` and `DELETE` operations by the application database account.
NFR-09: `[Team Target]` Under stress tests of 50 parallel reservation threads targeting the same resource or task, database-level pessimistic row locking (`SELECT ... FOR UPDATE` or ORM pessimistic locking) shall guarantee a 0% double-booking rate (0 race conditions).
NFR-10: `[Team Target]` Every failed or timed-out resource-package reservation shall execute a 100% atomic SQL rollback, committing zero task or resource changes and leaving changes committed by concurrent transactions intact.
NFR-11: `[Team Target]` The server shall enforce request idempotency (via client-generated request ID) on resource-reservation and task-completion requests so duplicate submissions arriving within ≤ 2.0 seconds produce 0 duplicate reservations or spurious state-transition errors.
NFR-12: `[Team Target]` If the server is offline across a scheduled `00:00:00 UTC` end-of-month SLA report trigger, the scheduler shall detect and execute any missed monthly report within ≤ 5.0 minutes of server restart (0 skipped monthly periods); for periods or incident types with 0 qualifying completed incidents, the SLA engine shall output `N/A` with 0 unhandled exceptions (such as `NullPointerException` or division-by-zero).
NFR-13: `[Team Target]` Upon a transient LAN disconnection, the JavaFX client shall automatically attempt reconnection within ≤ 5.0 seconds (up to 5 consecutive attempts) and receive a synchronized snapshot of open incidents, tasks, and reserved resources; upon graceful shutdown, the server shall complete in-flight SQL transactions within ≤ 10.0 seconds before closing client sockets (0 partially committed transactions), and recover on cold startup within ≤ 30.0 seconds.
NFR-14: `[Team Target]` A trained `Dispatcher` shall initiate and confirm a manual or automatic resource reservation in ≤ 3 clicks, a `Field Commander` shall report task completion in ≤ 2 clicks, and a `System Administrator` shall generate an on-demand SLA report in ≤ 3 clicks.
NFR-15: `[Team Target]` Upon any failed concurrent reservation, lock timeout, blocked predecessor check, invalid package validation, unavailable/faulty resource check, or unauthorized request, the client shall display an explicit human-readable error message naming the conflicting resource ID or failure reason within ≤ 1.0 second of receiving the server response.
NFR-16: `[Team Target]` The live incident status screen and SLA reporting interface shall distinguish all five task states (`Blocked`, `Ready`, `Active`, `Completed`, `Cancelled`) and overdue tasks using both distinct colors and explicit text/icon badges compliant with WCAG 2.1 Level AA (contrast ratio ≥ 4.5:1), rendering cleanly across resolutions from 1280×720 to 3840×2160 with 0 clipped action buttons or overlapping labels.
NFR-17: `[Team Target]` All timestamps used for SLA calculation, end-of-month report scheduling, and audit logging shall rely strictly on a single NTP-synchronized server clock (drift ≤ 1.0 second) with 0 reliance on client workstation clocks, persisted in UTC and formatted on the UI with an explicit timezone/UTC offset (`YYYY-MM-DD HH:MM:SS UTC±HH:MM`).
NFR-18: `[Team Target]` The automatic resource allocation module shall implement an `AllocationStrategy` interface such that adding and activating a new allocation criterion requires creating 1 new strategy class and modifying ≤ 1 factory/registration point, with 0 modifications to existing strategy implementations.
NFR-19: `[Team Target]` The reporting subsystem shall implement an extensible `ReportGenerator` interface such that adding and activating a new report type requires creating 1 new report class and modifying ≤ 1 registration point, with 0 modifications to existing report implementations.
NFR-20: The system shall enforce a strict 4-layer separation between JavaFX GUI, OCSF Communication, Server Business Logic, and SQL/ORM Persistence, with 0 direct SQL or ORM imports inside JavaFX GUI controller or view classes.

## Requirements Traceability Matrix

| Requirement ID | Use Case | Acceptance Test ID |
|---|---|---|
| FR-01 | [TBD] | AT-01, AT-12, AT-14, AT-15, AT-18 |
| FR-02 | [TBD] | AT-01, AT-02, AT-03, AT-06, AT-10 |
| FR-03 | [TBD] | AT-09 |
| FR-04 | [TBD] | AT-10 |
| FR-05 | [TBD] | AT-01, AT-08 |
| FR-06 | [TBD] | AT-01, AT-08 |
| FR-07 | [TBD] | AT-01 |
| FR-08 | [TBD] | AT-01, AT-02, AT-03, AT-04, AT-09 |
| FR-09 | [TBD] | AT-01 |
| FR-10 | [TBD] | AT-02, AT-03, AT-04, AT-09 |
| FR-11 | [TBD] | AT-05, AT-06, AT-07 |
| FR-12 | [TBD] | AT-06, AT-07 |
| FR-13 | [TBD] | AT-05, AT-06, AT-07 |
| FR-14 | [TBD] | AT-13, AT-14 |
| FR-15 | [TBD] | AT-13 |
| FR-16 | [TBD] | AT-01, AT-02, AT-03, AT-06, AT-13 |
| FR-17 | [TBD] | AT-15, AT-17, AT-18 |
| FR-18 | [TBD] | AT-15 |
| FR-19 | [TBD] | AT-16 |
| FR-20 | [TBD] | AT-01, AT-13, AT-15, AT-17, AT-19 |
| NFR-01 | [TBD] | AT-01, AT-05 |
| NFR-02 | [TBD] | AT-04, AT-06, AT-07 |
| NFR-03 | [TBD] | AT-13 |
| NFR-04 | [TBD] | AT-13 |
| NFR-05 | [TBD] | AT-15, AT-17, AT-20 |
| NFR-06 | [TBD] | AT-12, AT-14, AT-18 |
| NFR-07 | [TBD] | AT-19 |
| NFR-08 | [TBD] | AT-19 |
| NFR-09 | [TBD] | AT-02, AT-03, AT-06 |
| NFR-10 | [TBD] | AT-02, AT-03, AT-04, AT-07, AT-09 |
| NFR-11 | [TBD] | AT-11 |
| NFR-12 | [TBD] | AT-16, AT-17 |
| NFR-13 | [TBD] | AT-21 |
| NFR-14 | [TBD] | AT-01, AT-05, AT-13, AT-15 |
| NFR-15 | [TBD] | AT-02, AT-03, AT-04, AT-07, AT-08, AT-09, AT-12, AT-14, AT-18 |
| NFR-16 | [TBD] | AT-20 |
| NFR-17 | [TBD] | AT-15, AT-17 |
| NFR-18 | [TBD] | AT-22 |
| NFR-19 | [TBD] | AT-22 |
| NFR-20 | [TBD] | AT-23 |

## Acceptance Tests

### AT-01: Manual Resource Package Reservation — Uncontested Happy Path
- **Given** a `Dispatcher` is authenticated on the live incident status screen under normal load (≤ 50 concurrent workstations), and task "Debris Removal" is in `Ready` state (all predecessors `Completed`) requiring 1 `Field Commander` (`FC-01`), 1 `Heavy Equipment Operator` (`OP-04`), 1 `Bulldozer` (`EQ-BD-02`), and 1 `Medic` (`MED-07`), all currently `Available` in SQL
- **When** the `Dispatcher` selects the package and confirms manual reservation in ≤ 3 clicks
- **Then** the server executes an ACID SQL transaction with pessimistic row locking on the task and requested resources and authoritative in-transaction verification within ≤ 1.5 seconds (measured from server request receipt to commit and response dispatch), updates all four resources to `Reserved for Task`, transitions the task to `Active`, records an immutable audit log entry with the `Dispatcher`'s ID, and broadcasts the state change via OCSF Server Push.

### AT-02: Manual Resource Reservation — Shared-Resource Contention and Atomic Rollback
- **Given** 50 parallel reservation threads (including two `Dispatcher` workstations competing at the same millisecond) simultaneously request `Bulldozer EQ-BD-02` for separate `Ready` tasks (`Task-1` and `Task-2`), and every resource in all competing packages (including `EQ-BD-02`, `FC-03`, and all other requested items) is initially valid and `Available`
- **When** the competing manual reservation transactions execute concurrently on the server
- **Then** pessimistic row locking grants `EQ-BD-02` to **exactly one** succeeding transaction (0% double-booking rate), while every losing transaction rolls back completely—committing zero task or resource changes and without undoing the winning transaction's committed changes—leaves its own task in `Ready`, notifies the losing `Dispatcher`(s) with an error naming `EQ-BD-02` within ≤ 1.0 second of server response, and pushes the winning state change via OCSF Server Push.

### AT-03: Manual Resource Reservation — Same-Task Contention with Disjoint Packages
- **Given** two `Dispatcher` workstations simultaneously view `Task-1` as `Ready` and submit two completely disjoint, valid resource packages for `Task-1`: Package A (`FC-01`, `OP-01`, `EQ-BD-01`, `MED-01`) and Package B (`FC-02`, `OP-02`, `EQ-BD-02`, `MED-02`), where all eight resources are initially `Available`
- **When** both `Dispatcher` workstations submit manual reservations for `Task-1` concurrently
- **Then** pessimistic row locking on `Task-1` and the requested resources allows **exactly one** request to succeed (transitioning `Task-1` to `Active` and reserving its package), while the second request observes inside the transaction that `Task-1` is no longer `Ready`, rolls back completely—committing zero reservations from its disjoint package and leaving the winning transaction's committed reservation intact—notifies the losing `Dispatcher` within ≤ 1.0 second of server response that `Task-1` is already `Active`, and refreshes the screen via OCSF Server Push.

### AT-04: Manual Resource Reservation — Lock-Wait Timeout (Deadlock Prevention)
- **Given** a `Ready` task requests equipment `EQ-BD-05`, and another database session holds a conflicting pessimistic row lock on `EQ-BD-05` for longer than 3.0 seconds
- **When** a `Dispatcher` submits the reservation request
- **Then** the database lock-wait times out at ≤ 3.0 seconds, the server completes the full SQL rollback—committing zero task or resource changes and leaving concurrent committed changes intact—dispatches the failure response within ≤ 3.5 seconds from server request receipt, and displays an explicit human-readable timeout error within ≤ 1.0 second of receiving the server response.

### AT-05: Automatic Resource Allocation — Uncontested Happy Path
- **Given** a `Ready` task requires 1 `Field Commander`, 1 `Medic`, and 1 `Ambulance`, with multiple matching `Available` resources across stations under normal uncontested load
- **When** the `Dispatcher` selects the "closest geographic proximity" criterion and confirms automatic allocation in ≤ 3 clicks
- **Then** the server selects the closest available package, acquires pessimistic row locks on the task and selected resources, verifies status inside the SQL transaction, commits the atomic reservation (`Ready` to `Active`) within ≤ 1.5 seconds from server request receipt, and notifies the `Dispatcher`.

### AT-06: Automatic Resource Allocation — Contention Fallback to Second Alternative
- **Given** a `Ready` task requests automatic allocation using "closest geographic proximity" where the three closest matching ambulances in preference order are `EQ-AMB-01` (1st choice), `EQ-AMB-02` (1st fallback), and `EQ-AMB-03` (2nd fallback, `Available`), and concurrent reservations claim both `EQ-AMB-01` and `EQ-AMB-02` before this request acquires locks
- **When** the server executes automatic allocation alongside unrelated active user requests
- **Then** the initial selection (`EQ-AMB-01`) and 1st fallback (`EQ-AMB-02`) detect contention, the server automatically advances to the 2nd fallback (`EQ-AMB-03`) without `Dispatcher` resubmission (adding ≤ 500 milliseconds per fallback retry and ≤ 100 milliseconds delay to unrelated requests), locks and reserves the complete package with `EQ-AMB-03` atomically (`Ready` to `Active`) within ≤ 3.5 seconds from server request receipt, and pushes updates via OCSF Server Push.

### AT-07: Automatic Resource Allocation — Exhausted Resource Pool Failure
- **Given** a `Ready` task requires 1 `Bulldozer`, the initial candidate and up to two fallback `Bulldozer` candidates are `Reserved for Task` or `Faulty/In Maintenance`, and no other matching `Available` `Bulldozer` exists in the resource pool
- **When** the server completes the initial attempt and fallback checks
- **Then** the system aborts allocation within ≤ 3.5 seconds from server request receipt with an atomic rollback (0 partial reservations, leaving concurrent committed transactions intact), keeps the task in `Ready`, and displays an explicit human-readable error within ≤ 1.0 second of server response stating no available `Bulldozer` remains in the pool.

### AT-08: Resource Reservation Validation — Structural Pre-Checks
- **Scenario Outline:** Reject invalid task state or invalid `Field Commander` package count
    - **Given** a `Dispatcher` opens a task with `<Precondition>`
    - **When** the `Dispatcher` submits a reservation request
    - **Then** the system rejects the request before acquiring resource locks, commits zero changes, and displays an explicit validation error naming `<ExpectedMessage>` within ≤ 1.0 second.

| Case | Precondition | ExpectedMessage |
|---|---|---|
| Blocked task | Task is in `Blocked` state because predecessor `Task-A` is still `Active` | `Task-A` (`Blocked` state) |
| Zero commanders | `Ready` task package contains 0 `Field Commanders` | Missing required `Field Commander` (must be exactly 1) |
| Multiple commanders | `Ready` task package contains 2 `Field Commanders` | Multiple `Field Commanders` (must be exactly 1) |

### AT-09: Resource Reservation Validation — Authoritative In-Transaction Status Verification
- **Given** a `Dispatcher` submits a reservation for a `Ready` task containing `MED-08` and `EQ-GEN-01` that appeared `Available` in the UI/pre-check, while concurrently `MED-08` transitions to `Unavailable` (or `EQ-GEN-01` transitions to `Faulty/In Maintenance`) before row locks are acquired
- **When** the server acquires pessimistic row locks inside the SQL transaction and performs authoritative status verification
- **Then** the in-transaction check detects the updated `Unavailable` or `Faulty/In Maintenance` status, rolls back the transaction completely—committing zero changes and leaving concurrent committed transitions intact—and returns an explicit error displayed within ≤ 1.0 second naming `MED-08` or `EQ-GEN-01`.

### AT-10: Field Commander Dual Status Enforcement
- **Given** `Field Commander FC-01` is currently `Reserved for Task` on `Active` task `Task-A`
- **When** `FC-01` logs into the system and interacts with `Task-A` while a `Dispatcher` attempts to reserve `FC-01` for `Task-B`
- **Then** `FC-01` continues operating normally in the system for `Task-A`, while the reservation of `FC-01` for `Task-B` is rejected leaving `Task-B` in `Ready` and `FC-01` reserved solely to `Task-A`.

### AT-11: Request Idempotency on Duplicate Submissions
- **Given** a `Dispatcher` (or appointed `Field Commander`) submits a valid reservation (or task-completion) request with a unique client request ID, and an identical duplicate request with the same request ID arrives within ≤ 2.0 seconds due to accidental double-click or network retry
- **When** the server processes both submissions
- **Then** the server executes the state transition once and returns the idempotent acknowledgment for the duplicate submission, producing 0 duplicate reservations and 0 spurious state-transition errors.

### AT-12: Unauthorized Reservation Role Rejection
- **Given** a user without the `Dispatcher` role (e.g., `Field Commander` or `Logistics Officer`) is authenticated on a workstation
- **When** the user sends a manual or automatic resource-package reservation OCSF request to the server
- **Then** server-side RBAC rejects the request within ≤ 100 milliseconds without acquiring locks or modifying any resource/task state, and displays an explicit authorization error within ≤ 1.0 second of server response.

### AT-13: Task Completion Reporting — Happy Path, Faulty-Equipment Preservation, Percentile Latency & Server Push
- **Given** `Field Commander FC-01` is authenticated and appointed to `Active` task `Task-A` (associated with `FC-01` and `MED-02` in `Reserved for Task`, plus `EQ-BD-09` which transitioned to `Faulty/In Maintenance` mid-task), and `Task-B` is `Blocked` with `Task-A` as its sole open predecessor; additionally, a benchmark series of 100 consecutive task-completion/push events is measured under normal load
- **When** `FC-01` reports completion of `Task-A` in ≤ 2 clicks (and across the 100-event measurement series)
- **Then** `Task-A` transitions to immutable `Completed`, `FC-01` and `MED-02` return to `Available` while `EQ-BD-09` retains `Faulty/In Maintenance`, `Task-B` promotes from `Blocked` to `Ready`, an immutable audit entry is recorded; and across the 100-event series the 95th-percentile server commit latency is ≤ 1.0 second and the 95th-percentile OCSF Server Push delivery latency is ≤ 1.0 second (with 0 client polling requests and ≤ 64 KB average delta payload).

### AT-14: Task Completion — Unauthorized Commander and Immutable Completed State
- **Given** `Field Commander FC-02` is NOT appointed to `Active` task `Task-A` (assigned to `FC-01`), and `Task-C` is already in `Completed` state
- **When** `FC-02` submits a completion request for `Task-A`, or any user attempts to report completion on `Task-C`
- **Then** server-side RBAC rejects `FC-02` within ≤ 100 milliseconds with an explicit authorization error displayed within ≤ 1.0 second, `Task-A` remains `Active` with unchanged resources, and `Task-C` remains immutably `Completed`.

### AT-15: SLA Report Generation — On-Demand Monthly Happy Path & Month Attribution
- **Given** 10,000 historical task records exist across incident types, including incidents whose last task reached a final state (`Completed` or `Cancelled`) in October 2026 (UTC), `Cancelled` false-alarm incidents, incidents with zero `Completed` tasks, and incidents still open at month-end
- **When** an authenticated `System Administrator` selects October 2026 and requests an on-demand SLA report in ≤ 3 clicks
- **Then** the server computes per-type average elapsed time (from incident opening timestamp to the latest final-state task transition timestamp) including only qualifying October 2026 incidents (excluding `Cancelled`, zero-completed-task, and open incidents), records an audit entry with the `System Administrator`'s ID, and renders the report within ≤ 3.0 seconds using NTP-synchronized server timestamps formatted with explicit UTC offset (`YYYY-MM-DD HH:MM:SS UTC±HH:MM`).

### AT-16: SLA Report Generation — Empty Dataset Handling
- **Given** the `System Administrator` requests an SLA report for a past month in which zero qualifying completed incidents exist for one or more incident types
- **When** the reporting engine executes the SLA aggregation query
- **Then** the system completes with 0 unhandled exceptions (such as `NullPointerException` or division-by-zero) and outputs `N/A` (never `0`) for every category with zero qualifying completed incidents.

### AT-17: SLA Report Generation — Scheduled Month-End Execution, `SYSTEM` Audit & Missed Recovery
- **Given** qualifying completed incident records exist for the preceding calendar month according to the NTP-synchronized server clock (drift ≤ 1.0 second), and the server was offline across the scheduled `00:00:00 UTC` trigger on the first day of the following month
- **When** the server starts up after the missed `00:00:00 UTC` trigger window
- **Then** the scheduler detects and executes the missed monthly SLA report for the preceding calendar month within ≤ 5.0 minutes of restart (completing batch computation across up to 10,000 tasks within ≤ 10.0 seconds), persists the report with 0 skipped periods, and writes an immutable audit log entry with acting user identifier `SYSTEM`.

### AT-18: SLA Report Generation — Unauthorized Role Rejection
- **Given** a `Dispatcher` (non-`System Administrator`) is authenticated on a workstation
- **When** the `Dispatcher` submits an OCSF request to generate or view an SLA report
- **Then** server-side RBAC rejects the request within ≤ 100 milliseconds, returns no report data, and displays an explicit authorization error within ≤ 1.0 second of server response.

### AT-19: Security Controls — TLS, Credential Protection, SQLi Prevention & Append-Only Audit
- **Given** workstations communicate over LAN TCP/IP (OCSF), user passwords and database credentials are stored at rest, SQL/ORM queries execute server-side, and events are logged to the audit table
- **When** network traffic, stored credentials, logs/error dialogs, SQL-injection inputs, and `UPDATE`/`DELETE` statements from the application DB account on the audit table are tested
- **Then** 100% of OCSF traffic uses TLS 1.2/1.3 (AES-256), user passwords are stored as salted BCrypt hashes, database credentials at rest are encrypted using AES-256 with 0 plaintext secrets, stack traces, or SQL schemas in logs or error dialogs, 100% parameterized queries block SQL injection, and DB privileges reject 100% of `UPDATE` and `DELETE` operations on the audit log.

### AT-20: Live Status Screen & SLA UI — Scalability, Accessibility & Responsive Layout
- **Given** 50 workstations are concurrently connected and a live incident contains 200 tasks across all five states including an overdue `Active` task
- **When** a `Dispatcher` or appointed `Field Commander` opens the live status screen (or `System Administrator` opens the SLA report) across resolutions from `1280×720` to `3840×2160`
- **Then** the screen loads and renders within ≤ 2.0 seconds with 0 clipped action buttons or overlapping labels, and all task states and overdue indicators are distinguishable using both distinct colors and explicit text/icon badges compliant with WCAG 2.1 Level AA (contrast ratio ≥ 4.5:1).

### AT-21: Resilience — Client LAN Reconnection, Graceful Shutdown & Cold Startup
- **Given** a JavaFX client is connected to the running server and an in-flight SQL reservation transaction is executing
- **When** a transient LAN drop occurs on the client, and subsequently the server receives a graceful shutdown signal followed by a cold restart
- **Then** the client attempts reconnection within ≤ 5.0 seconds (up to 5 attempts) and receives a synchronized snapshot of open incidents, tasks, and reserved resources; on shutdown the server completes in-flight SQL transactions within ≤ 10.0 seconds before closing client sockets (0 partially committed transactions); and cold startup recovers within ≤ 30.0 seconds.

### AT-22: Extensibility Verification — Adding New Allocation Strategy and New Report Type
- **Given** the existing automatic allocation and reporting subsystems are operational
- **When** a developer adds 1 new strategy class implementing `AllocationStrategy` (registered at 1 factory/registry point) and 1 new report class implementing `ReportGenerator` (registered at 1 registry point) without modifying any existing strategy or report classes
- **Then** the project compiles cleanly, selecting the new strategy executes the new allocation criterion during automatic allocation, and invoking the new report generator produces the new report without breaking existing strategies or the SLA report.

### AT-23: Architectural Verification — 4-Layer Separation
- **Given** the CrisisSync multi-layer codebase (JavaFX GUI, OCSF Communication, Server Business Logic, SQL/ORM Persistence)
- **When** an automated static dependency check (or package import audit) scans all JavaFX GUI controller and view classes
- **Then** 0 direct SQL (`java.sql.*`, `javax.sql.*`) or ORM imports are present in the GUI layer.