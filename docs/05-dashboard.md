Dashboard

Purpose: Provide a practical interface for reviewing security events and alerts collected by LogRadar.

1. Dashboard Overview

The React dashboard is the investigation layer of LogRadar.

Instead of looking directly at Linux log files or FastAPI console output, the dashboard presents the processed events in one place.

The current dashboard focuses on:

Security events

High-severity activity

Failed SSH attempts

Active hosts

Detection rules

Correlated alerts

Search and filtering

The dashboard is intentionally focused on the data produced by the monitoring pipeline rather than trying to reproduce a full commercial SIEM.

2. Dashboard Flow

PostgreSQL
     ↓
FastAPI API
     ↓
React
     ↓
Dashboard

The frontend requests processed data from the backend.

For example:

GET /api/logs
GET /api/alerts

The backend returns structured JSON and React uses that data to update the interface.

3. Main Dashboard Sections

The current interface is divided into a few practical areas.

┌─────────────────────────────────────────────────────────┐
│ LogRadar                         Search      Theme      │
├──────────────┬──────────────────────────────────────────┤
│              │                                          │
│ Events       │ Security Events                          │
│ Alerts       │                                          │
│ Agents       │ Statistics                               │
│ Detection    │                                          │
│ Rules        │ Event Stream                             │
│              │                                          │
└──────────────┴──────────────────────────────────────────┘

The sidebar is used for navigation while the main area contains the current security data.

4. Security Statistics

The dashboard shows a few counters at the top.

Example:

Events          25
High severity    5
Failed SSH       5
Active hosts     1

Events

Total number of events currently available to the dashboard.

High Severity

Number of events that have a high severity level.

Failed SSH

Number of detected failed SSH authentication events.

Active Hosts

Number of unique monitored hostnames present in the event data.

These counters provide a quick overview before looking at individual events.

5. Event Stream

The event stream displays individual processed events.

Example:

Severity

Detection

Rule

Source

Host

User / IP

Time

7 High

Failed SSH Login

SSH-001

auth.log

ip-172-31-28-255

baduser / 152.57.18.51

11:50:16 PM

3 Info

Successful SSH Login

SSH-002

auth.log

ip-172-31-28-255

ubuntu / 152.57.19.200

11:49:02 PM

4 Medium

Sudo Command

SUDO-001

auth.log

ip-172-31-28-255

ubuntu

11:47:31 PM

The original log message is also available for investigation.

6. Severity Display

Severity is displayed using different visual indicators.

The current project uses:

1  → Information
3  → Successful authentication
4  → Medium / privileged activity
7  → High / failed SSH
10 → Critical / correlated brute-force alert

For example:

[ 1 ] Info
[ 4 ] Medium
[ 7 ] High
[10 ] Critical

The visual severity indicator helps an analyst quickly identify events that deserve attention.

7. Event Details

An event contains more than just its severity.

A typical event can contain:

Timestamp
Hostname
Source log
Username
Source IP
Event type
Rule ID
Detection name
Severity
MITRE mapping
Original message

Example:

Detection : Failed SSH Login - Invalid User
Rule      : SSH-001
Severity  : 7
Source    : /var/log/auth.log
Host      : ip-172-31-28-255
User      : baduser
Source IP : 152.57.18.51
Time      : 11:50:16 PM
MITRE     : T1110

This makes the dashboard useful for basic investigation rather than only showing a raw log line.

8. Alerts

Alerts are different from normal events.

An event may represent:

One failed SSH login

An alert represents a pattern:

5 failed SSH logins
from the same IP
within the configured time window

Example alert:

┌─────────────────────────────────────────────┐
│ CRITICAL                                    │
│ SSH Brute Force Detected                   │
│                                             │
│ Source IP : 152.57.18.51                   │
│ Attempts  : 5                               │
│ Window    : 5 minutes                      │
│ Rule      : CORR-SSH-001                    │
│ MITRE     : T1110                           │
└─────────────────────────────────────────────┘

The original events remain available so the alert can be investigated.

9. Search

The dashboard provides a search field for quickly finding events.

The search can be useful for values such as:

baduser
152.57.18.51
SSH-001
Failed SSH
auth.log

The frontend filters the loaded event data based on the search value.

This is useful when the event stream contains many entries.

10. Severity Filters

The event stream can be narrowed using severity filters.

Example options:

All severities
High & Critical
Medium+

This allows an analyst to move from the complete event stream to higher-priority events without manually scanning every row.

11. Live Updates

The current dashboard uses periodic API requests to update the event stream.

The frontend requests the latest data approximately every 1.5 seconds.

Example backend output:

127.0.0.1:39345 - "GET /api/logs HTTP/1.1" 200
127.0.0.1:39345 - "GET /api/logs HTTP/1.1" 200
127.0.0.1:39345 - "GET /api/logs HTTP/1.1" 200

A successful 200 response means the API returned the requested data.

This gives the dashboard a near-real-time view without requiring a WebSocket connection in the current implementation.

12. Light and Dark Mode

The dashboard supports both light and dark themes.

The theme is a UI preference and does not affect the security data or detection logic.

Example:

Light Mode
──────────
White background
Dark text
Light borders

Dark Mode
─────────
Dark background
Light text
Dark borders

The same security events are displayed in both modes.

13. Agents

The Agents section represents the machines sending telemetry to LogRadar.

The current project has been tested with a Linux VPS agent.

Basic agent information can include:

Hostname
Status
Log sources
Last activity

Example:

Host: ip-172-31-28-255
Status: Connected
Sources:
  /var/log/auth.log
  /var/log/syslog

14. Detection Rules

The Detection Rules section provides visibility into the rules used by the detection engine.

Current examples:

SSH-001
Failed SSH Login

SSH-002
Successful SSH Login

SUDO-001
Sudo Command Executed

CORR-SSH-001
SSH Brute Force Detected

This makes it easier to understand why an event received a particular detection and severity.

15. Example Investigation

A basic investigation can start from the event stream.

Step 1 — Notice a high-severity event

Severity: 7
Detection: Failed SSH Login

Step 2 — Check the source

Source IP: 152.57.18.51

Step 3 — Search the same IP

The search field can be used to find other events from:

152.57.18.51

Step 4 — Check for a pattern

If multiple failed events are found:

1
2
3
4
5

within the configured time window, the correlation engine can create:

SSH Brute Force Detected

Step 5 — Review the alert

The analyst can then review:

Source IP
Attempts
Time window
Rule
MITRE mapping
Original events

This provides a basic investigation workflow from event → pattern → alert.

16. Dashboard-to-Backend Flow

The complete dashboard flow is:

                React Dashboard
                       │
                       │ GET /api/logs
                       ▼
                 FastAPI API
                       │
                       ▼
                 PostgreSQL
                       │
                       ▼
              Processed Events
                       │
                       ▼
                React renders
                       │
                       ▼
               Security Event

For alerts:

React Dashboard
       │
       │ GET /api/alerts
       ▼
   FastAPI API
       │
       ▼
   Alert records
       │
       ▼
   React Alert UI

17. Current Dashboard Scope

The current dashboard is designed around the monitoring workflow already implemented in LogRadar.

It provides:

Event visibility

Severity classification

Detection information

Source and host information

Search

Severity filtering

Near-real-time updates

Correlated alert visibility

Agent status

Detection rule visibility

Light and dark themes

The dashboard is intended to make the output of the backend easier to investigate rather than replace the underlying detection and correlation pipeline.
