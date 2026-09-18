# Secrets Rotation Watchdog

Secrets Rotation Watchdog is an automated monitoring and alerting workflow built in N8N that tracks the lifecycle of API keys, certificates, passwords, and tokens stored in a local credential inventory. Operating on a scheduled cron trigger, the workflow scans each secret, calculates the remaining validity against defined expiration thresholds, and classifies entries into distinct urgency states. It automatically notifies credential owners ahead of expiration, deduplicates repeated alerts, and dispatches parallel Slack notifications. When expired credentials remain unrotated past grace periods, the system triggers tiered escalations to administrators and security leads while persisting every operational event to an audit log.

## How it works

The automation initiates execution on a daily cron schedule or via manual invocation. It reads the credential store from `data/mock_credentials.json`, extracts the JSON payload, and splits the array into individual records for processing. A dedicated classification node evaluates the expiration date of each credential against the current date, computing both days remaining and days overdue. Based on these calculations, credentials are assigned a status of `ok`, `warning`, `expiring_soon`, or `expired`.

Once classified, records flow through routing and conditional evaluation paths. Credentials with valid lifecycles require no operational action. For credentials in `warning` or `expiring_soon` status, the workflow verifies whether a notification for the current state was already dispatched. If not previously sent, it generates and delivers a formatted HTML notification email directly to the credential owner.

When a credential enters the `expired` state, the workflow dispatches an initial expiration notice to the owner and evaluates overdue duration against three distinct escalation tiers. Tier 1 (3 days overdue) sends an escalation email to the administrative contact. Tier 2 (7 days overdue) issues a critical notice to the administrative team and posts an alert block to Slack via webhook. Tier 3 (14 days overdue) escalates to the security lead for revocation review and triggers an additional Slack alert. 

Following notification and escalation handling, the workflow aggregates all processed records in a snapshot builder node. This node updates internal tracking flags, records timestamps, prepares updated binary files, and synchronously writes the updated credential state back to `data/mock_credentials.json` and execution event records to `data/audit_log.json`.

## Credential classification

| Status Label | Condition | Action Triggered |
| :--- | :--- | :--- |
| `ok` | More than 30 days until expiry | No action required. Resets notification and escalation flags if credential was rotated. |
| `warning` | 8 to 30 days until expiry | Sends standard 30-day warning email to credential owner if not previously notified for this status. |
| `expiring soon` | 1 to 7 days until expiry | Sends urgent 7-day expiration notice to credential owner if not previously notified for this status. |
| `expired` | 0 or fewer days until expiry | Sends expired notice to owner; evaluates escalation tiers (Tier 1: 3+ days, Tier 2: 7+ days, Tier 3: 14+ days) for admin emails and Slack alerts. |

## Project structure

```
Secrets-Watchdog/
├── Workflow/
│   └── Secrets-Watchdog_v1.1.json
├── data/
│   ├── audit_log.json
│   └── mock_credentials.json
├── .gitignore
└── README.md
```

## mock_credentials.json

The `data/mock_credentials.json` file is a template only. Delete all example entries and populate the file with your own credentials. Each record in the provided example dataset uses placeholder email addresses named `EMAIL_1` through `EMAIL_8`. To locate and replace them all, search for `EMAIL_` using Ctrl+F.

```json
{
  "id": "cred-001", // Unique identifier for the credential
  "name": "prod-payments-stripe-api-key", // Descriptive name of the secret or token
  "type": "api_key", // Credential type (e.g., api_key, token, cert, password)
  "owner_email": "EMAIL_1", // Email address of the assigned credential owner
  "expiry_date": "2026-12-04", // Expiration date in ISO format (YYYY-MM-DD)
  "last_rotated": "2026-09-05", // Date the credential was last rotated (YYYY-MM-DD)
  "status": "ok", // Lifecycle status (ok, warning, expiring_soon, expired, unknown)
  "escalation_level": 0, // Current recorded escalation level (0 to 3)
  "notify_sent": false, // Boolean flag indicating if an owner notification was sent
  "escalated_to_admin": false, // Boolean flag indicating if admin escalation was triggered
  "should_notify": false, // Runtime flag computed by classifier for owner notification
  "escalation_tier": 0, // Runtime target escalation tier (0: none, 1: day 3, 2: day 7, 3: day 14)
  "should_escalate": false, // Runtime flag indicating if escalation notification should fire
  "last_notified_status": null, // Last lifecycle status for which notification was dispatched
  "revocation_flagged": false, // Boolean flag indicating credential is recommended for revocation (tier 3)
  "days_left": 85, // Days remaining until expiration (negative if overdue)
  "days_since_expiry": 0 // Number of days past expiration (0 if still valid)
}
```

## Configuration

Personal credentials and environment-specific endpoints are not included in this repository. Every location in the workflow definition requiring a real value is marked with the comment `CHANGE THIS`. To configure the workflow, search for `CHANGE THIS` using Ctrl+F inside `Workflow/Secrets-Watchdog_v1.1.json` and replace each occurrence according to the following inventory:

* `Your Email address. CHANGE THIS` -- appears 6 times -- the sender or owner email used in notification nodes
* `Admin / security team Email address. CHANGE THIS` -- appears 1 time -- the escalation target for overdue credentials
* `Security lead Email address. CHANGE THIS` -- appears 1 time -- secondary escalation recipient
* `Your Slack webhook URL. CHANGE THIS` -- appears 1 time -- the incoming webhook URL for Slack alerts
* `Your SMTP ID code. CHANGE THIS` -- appears 6 times -- the SMTP credential ID used by N8N email nodes

## Prerequisites

1. N8N installed locally or on cloud
2. An SMTP account (Gmail or other) configured as an N8N credential
3. A Slack workspace with an incoming webhook set up
4. Node.js if running N8N via npm

## Setup

1. Clone the repository to your local system or N8N host.
2. Import `Workflow/Secrets-Watchdog_v1.1.json` into your N8N instance.
3. Configure the SMTP credential in N8N.
4. Replace all `CHANGE THIS` values in the imported workflow.
5. Replace or build your own `data/mock_credentials.json` file.
6. Activate the workflow to enable the scheduled execution trigger.

## IAM and security concepts covered

* Credential lifecycle management: tracking secrets from issuance to expiration and renewal, matching secret leasing models in HashiCorp Vault.
* Expiry threshold classification: segmenting credentials into proactive warning windows, analogous to rotation schedules in AWS Secrets Manager.
* Owner notification with deduplication: dispatching alerts only on status transitions to prevent notification fatigue, fulfilling access monitoring requirements under SOC 2 CC6.1.
* Tiered escalation policy: escalating unrotated credentials through defined administrative hierarchies, aligning with incident response guidelines in ISO 27001 A.12.1.
* Audit logging: recording execution parameters, status transitions, and actions to structured storage, satisfying audit trail controls under SOC 2 CC7.2 and ISO 27001 A.12.4.
* Idempotency in automated workflows: ensuring repeated schedule executions produce consistent state updates without duplicate side effects, essential for automated identity governance.

## Notes

The mock credential store in `data/mock_credentials.json` is intentionally simple and serves as a local data source for demonstration and testing. In a production deployment, this local file is designed to be replaced with direct API integrations to dedicated secret management systems such as HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault.

This workflow does not perform actual credential rotation. Its scope is strictly detection, classification, owner notification, policy escalation, and audit logging. Actual secret regeneration and consumer deployment must be executed through downstream automation or manual rotation procedures.

The audit log implementation records structured execution snapshots directly to `data/audit_log.json`. For every execution run, the workflow writes entry records detailing the timestamp, execution ID, credential ID, credential name, event type, actor name, escalation level, and operation result.
