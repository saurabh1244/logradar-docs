LogRadar Architecture

Overview

LogRadar is divided into a few simple components.

The main flow is:

Linux VPS↓LogRadar Agent↓FastAPI↓Log Parser↓Detection Rules↓PostgreSQL↓Correlation Engine↓React Dashboard

1. Linux VPS

The VPS is the machine being monitored.

LogRadar currently watches:

/var/log/auth.log

/var/log/syslog

For example, an SSH attempt can generate a log like:

2026-08-12T17:17:22.332098+00:00 ip-172-31-28-255 sshd-session[44055]:
Invalid user testuser123 from 152.57.19.200 port 46203

2. LogRadar Agent

The Python agent runs on the VPS.

Its job is to continuously watch the configured log files and send newly detected lines to the FastAPI backend.

Example agent output:

LogRadar Agent started.
Sending to https://<BACKEND-URL>/logs

[WATCHING] /var/log/auth.log
[WATCHING] /var/log/syslog

[LOG] /var/log/auth.log:
2026-08-12T17:17:22.332098+00:00 ip-172-31-28-255 sshd-session[44055]:
Invalid user testuser123 from 152.57.19.200 port 46203

[SENT] /var/log/auth.log | 200

3. FastAPI Backend

The agent sends each log entry to the FastAPI /logs endpoint.

The request contains:

{
  "hostname": "ip-172-31-28-255",
  "source": "/var/log/auth.log",
  "message": "Invalid user testuser123 from 152.57.19.200 port 46203"
}

FastAPI receives the event and passes it through the processing pipeline.

Example backend output:

[ip-172-31-28-255] /var/log/auth.log ->
Invalid user testuser123 from 152.57.19.200 port 46203

98.89.33.78:0 - "POST /logs HTTP/1.1" 200

4. Log Parser

The parser converts the raw log message into structured information.

For example:

Invalid user testuser123 from 152.57.19.200 port 46203

becomes:

event_type = ssh_failed
username   = testuser123
source_ip  = 152.57.19.200
port       = 46203

This structured data is then used by the detection rules.

5. Detection Engine

The detection engine checks the parsed event against the configured rules.

For example:

SSH failed
    ↓
SSH-001
    ↓
Severity: 7
    ↓
MITRE: T1110

A successful SSH login or sudo activity follows its own detection rule.

6. PostgreSQL

After processing, the event is stored in PostgreSQL.

The project currently uses Neon PostgreSQL.

The database keeps the events available even after the FastAPI process is restarted.

The main event information includes:

timestamp
hostname
source
message
event_type
username
source_ip
severity
rule_id
rule
MITRE information

7. Correlation Engine

Some detections require more than one event.

For example:

Failed SSH
Failed SSH
Failed SSH
Failed SSH
Failed SSH

If these attempts come from the same IP within the configured time window, the correlation engine can generate a brute-force alert.

Example:

[CORRELATION] 152.57.18.51 -> 1 failed SSH attempts
[CORRELATION] 152.57.18.51 -> 2 failed SSH attempts
[CORRELATION] 152.57.18.51 -> 3 failed SSH attempts
[CORRELATION] 152.57.18.51 -> 4 failed SSH attempts
[CORRELATION] 152.57.18.51 -> 5 failed SSH attempts

============================================================
SSH BRUTE FORCE DETECTED
============================================================
Source IP : 152.57.18.51
Attempts  : 5
Window    : 5 minutes
============================================================

8. React Dashboard

The React application reads the processed events and alerts from FastAPI.

The dashboard displays:

Security events

Severity

Detection rule

Source IP

Host

Timestamp

Original log

Correlated alerts

The frontend periodically requests the latest data from:

GET /api/logs
GET /api/alerts

Example FastAPI requests:

127.0.0.1:39345 - "GET /api/logs HTTP/1.1" 200
127.0.0.1:39345 - "GET /api/alerts HTTP/1.1" 200

Complete Event Flow

A failed SSH attempt therefore moves through the system like this:

SSH attempt
     ↓
Linux writes to auth.log
     ↓
LogRadar Agent detects new line
     ↓
POST /logs
     ↓
FastAPI receives event
     ↓
Parser extracts IP / username / event type
     ↓
Detection rule assigns severity
     ↓
Event stored in PostgreSQL
     ↓
Correlation checks previous events
     ↓
Brute-force alert generated if threshold is reached
     ↓
React dashboard displays the result

The important part of the architecture is that the system does not directly display the raw log and stop there. The log passes through collection, parsing, detection, storage and correlation before becoming a security event or alert.
