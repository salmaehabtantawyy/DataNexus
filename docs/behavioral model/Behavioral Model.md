# DataNexus — Behavioral Model

This document describes how the DataNexus system behaves in response to events, user interactions,
and internal processes. It captures the dynamic aspects of the system using four UML diagrams,
each serving a distinct purpose in understanding system behavior.

---

## Diagrams

| Diagram          | File               | What It Shows                                              |
|------------------|--------------------|------------------------------------------------------------|
| Use Case         | `use_case.svg`     | 6 actors · 13 use cases · include/extend relationships     |
| Sequence         | `sequence.svg`     | Complete validation run message flow · 7 participants      |
| Activity         | `activity.svg`     | Validation process workflow · decision points · alert fork |
| State Machine    | `state_machine.svg`| Validation run lifecycle · 5 states · all transitions      |

---

## Actors & System Boundaries

The following actors interact with DataNexus:

| Actor                          | Role                                                                                             |
|--------------------------------|--------------------------------------------------------------------------------------------------|
| Data Engineer (Primary)        | Configures validation rules, runs validations, reviews results via CLI, Dashboard, and API.      |
| Business Stakeholder           | Read-only access. Views the Dashboard for quality scores and trends. Receives reports.           |
| Apache Airflow (Scheduler)     | Triggers automated validation runs on schedule (every 6 hours). Acts as orchestrator.           |
| CI/CD System (GitHub Actions)  | Triggers validation as a quality gate on push. Blocks deployment if score < threshold (95%).    |
| Email / SMTP Server            | External notification system. Delivers email alerts on validation failures.                      |
| Slack API                      | External messaging service. Receives webhook notifications for real-time alert delivery.         |

---

## Use Case Diagram — Key Relationships

The 13 use cases form three interaction groups:

**Data Engineer interactions (direct):**
UC1 Configure Rules → UC2 Run Validation → UC3 View Results · UC7 Trigger via API · UC12 Acknowledge Alert

**Automated system interactions:**
UC8 Schedule Runs (Airflow) · UC9 Quality Gate (CI/CD)

**`<<include>>` relationships** — always executed as part of another use case:
- UC2 Run Validation `<<include>>` UC4 Profile Dataset
- UC6 Manage Alerts `<<include>>` UC10 Send Email
- UC6 Manage Alerts `<<include>>` UC11 Send Slack

**`<<extend>>` relationship** — conditionally triggered:
- UC2 Run Validation `<<extend>>` UC6 Manage Alerts *(only when quality score < threshold)*

---

## Sequence Diagram — Validation Run

The sequence diagram traces the complete flow when a validation is triggered. Key design decisions visible in the diagram:

**Asynchronous execution:** The API returns `202 Accepted` immediately after triggering the Airflow DAG. The caller then polls `GET /api/runs/:id` for the result. This prevents HTTP timeouts on large dataset runs.

**DAG task order:** Airflow executes 4 tasks in sequence — `profile_data` → `run_validations` → `send_alerts` (conditional) → `persist_results`. Each task calls the corresponding Core Engine component.

**Conditional alerting:** The Alert Manager is only invoked when the quality score falls below the configured threshold. It is not part of the happy path.

---

## Activity Diagram — Data Validation Process

The activity diagram models the full workflow from trigger to completion. Key decision points:

| Decision Point    | Yes path                        | No / Error path                    |
|-------------------|---------------------------------|------------------------------------|
| Config valid?     | Proceed to dataset profiling    | Log error → set status = ERROR     |
| More rules?       | Execute next check (loop)       | Exit loop → calculate score        |
| Score ≥ threshold?| End with status = PASS          | Trigger Alert Manager → FAIL       |

The loop over validation rules is intentional — the engine processes rules one by one so that each failure is recorded individually, giving the Data Engineer per-rule visibility in the results.

---

## State Machine — Validation Run

A Validation Run object transitions through these states:

| State     | Trigger                                       | Color code |
|-----------|-----------------------------------------------|------------|
| `PENDING` | Run triggered (API / Airflow / CLI / CI/CD)   | Teal       |
| `RUNNING` | Config validated successfully                 | Blue       |
| `PASS`    | All checks pass · score ≥ threshold           | Green      |
| `FAIL`    | Score < threshold · alerts fired              | Yellow     |
| `ERROR`   | Technical failure (DB down, source missing)   | Red        |

**Retry behavior:** Both `FAIL` and `ERROR` states allow a manual retry, which transitions the run back to `PENDING`. This is the only way a terminal state can be re-entered.

**Why these 5 states and not more:** `PASS` and `FAIL` are kept separate (rather than a single `COMPLETE`) so that dashboards, alerts, and CI/CD gates can filter on outcome without parsing scores. The `ERROR` state is separate from `FAIL` because it indicates a system problem, not a data quality problem, and requires a different response from the team.
