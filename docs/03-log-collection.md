Log Collection

Purpose: Collect new security-related log entries from the Linux VPS and forward them to the LogRadar backend.

1. Log Sources

LogRadar currently monitors two Linux log files:

Log file

Main purpose

/var/log/auth.log

Authentication and security-related activity

/var/log/syslog

General system and service activity

/var/log/auth.log

This is the most important source for the current security detections.

It can contain events such as:

SSH authentication failures

Invalid SSH users

Successful SSH logins

Sudo activity

User sessions

Password-related activity

Example:

2026-08-12T17:17:22.332098+00:00 ip-172-31-28-255 sshd-session[44055]:
Invalid user testuser123 from 152.57.19.200 port 46203

From this line, LogRadar can identify information such as:

Event    : SSH authentication failure
Username : testuser123
Source IP: 152.57.19.200
Port     : 46203

2. /var/log/syslog

syslog contains general system activity.

Examples include:

systemd services

cron jobs

background services

kernel messages

system activity

Example:

2026-08-12T17:17:01.289102+00:00 ip-172-31-28-255 CRON[44053]:
(root) CMD (cd / && run-parts --report /etc/cron.hourly)

Not every syslog entry is a security event.

For that reason, LogRadar first collects the event and then lets the processing and detection layer decide whether it is relevant.

3. Why Use an Agent?

The LogRadar Agent runs directly on the monitored Linux VPS.

Its responsibility is intentionally small:

Read Linux logs
      ↓
Detect new lines
      ↓
Create event payload
      ↓
Send to FastAPI

The agent does not make the final security decision.

This keeps collection separate from detection.

4. Watching the Log Files

The agent follows the configured files and waits for new entries.

When a new line appears, it is immediately picked up.

Example agent output:

LogRadar Agent started.
Sending to https://<BACKEND-URL>/logs

[WATCHING] /var/log/auth.log
[WATCHING] /var/log/syslog

[LOG] /var/log/auth.log:
2026-08-12T17:17:22.332098+00:00 ip-172-31-28-255 sshd-session[44055]:
Invalid user testuser123 from 152.57.19.200 port 46203

After collecting the line, the agent sends it to the backend.

[SENT] /var/log/auth.log | 200

HTTP 200 means the backend successfully accepted the request.

5. Event Payload

The agent sends a small JSON payload instead of sending the entire log file.

Example:

{
  "hostname": "ip-172-31-28-255",
  "source": "/var/log/auth.log",
  "message": "Invalid user testuser123 from 152.57.19.200 port 46203"
}

Fields

Field

Description

hostname

Identifies the monitored machine

source

Log file that generated the event

message

Original log line

The original message is preserved because it can be useful during investigation.

6. Sending to FastAPI

The agent sends each new event to:

POST /logs

Example backend output:

[ip-172-31-28-255] /var/log/auth.log ->
Invalid user testuser123 from 152.57.19.200 port 46203

98.89.33.78:0 - "POST /logs HTTP/1.1" 200

The flow is therefore:

Linux
  │
  │ writes new log
  ▼
auth.log / syslog
  │
  │ new line detected
  ▼
LogRadar Agent
  │
  │ POST /logs
  ▼
FastAPI

7. Handling Permission

Some Linux log files require elevated permissions to read.

For example:

/var/log/auth.log

may not be readable by a normal user.

When permission is denied, the agent reports it instead of silently failing.

Example:

[WATCHER ERROR] Permission denied: /var/log/auth.log

The agent can then be run with the required permissions on the monitored VPS.

8. Log Rotation

Linux does not keep writing to the same physical log file forever.

Log files can be rotated and replaced.

For example:

auth.log
auth.log.1
auth.log.2.gz

The agent therefore needs to detect when the active log file changes and reopen the new file.

This prevents the collector from continuing to watch an old file after rotation.

9. Collection vs Detection

One important design decision in LogRadar is keeping these responsibilities separate.

Collection

The agent answers:

"What new log entry appeared?"

Detection

The backend answers:

"Does this event represent something interesting?"

For example:

Collection
──────────
Invalid user baduser from 152.57.18.51

becomes:

Detection
─────────
Rule: SSH-001
Severity: High
Event: Failed SSH Login

This separation makes the agent simpler and allows the detection logic to change without changing the collector.

10. Current Collection Flow

┌──────────────────────┐
│      Linux VPS       │
│                      │
│  auth.log            │
│  syslog              │
└──────────┬───────────┘
           │
           │ new log line
           ▼
┌──────────────────────┐
│   LogRadar Agent     │
│                      │
│  Watch               │
│  Read                │
│  Send                │
└──────────┬───────────┘
           │
           │ POST /logs
           ▼
┌──────────────────────┐
│     FastAPI API      │
└──────────┬───────────┘
           │
           ▼
     Detection Pipeline

11. Example: Complete Collection

A controlled SSH attempt:

ssh baduser@<VPS-IP>

generates a Linux log:

Invalid user baduser from 152.57.18.51 port 1894

The agent detects it:

[LOG] /var/log/auth.log:
Invalid user baduser from 152.57.18.51 port 1894

Then sends it:

[SENT] /var/log/auth.log | 200

FastAPI receives it:

POST /logs HTTP/1.1" 200

The event can then continue into the parsing and detection stages.

12. Current Scope

The current collector is intentionally focused on two log sources:

/var/log/auth.log
/var/log/syslog

The design can later be extended to additional Linux log sources, but the current implementation keeps the collection pipeline small and easy to troubleshoot.
