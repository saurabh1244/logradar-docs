Detection Engine

Purpose: Convert collected Linux log messages into structured security events and assign a detection rule, severity, and MITRE ATT&CK context.

1. Where Detection Happens

The detection engine runs after the LogRadar Agent has collected a log and FastAPI has received it.

Linux Log
    ↓
LogRadar Agent
    ↓
POST /logs
    ↓
FastAPI
    ↓
Parser
    ↓
Detection Engine
    ↓
Structured Security Event

The collector only answers:

"What happened on the system?"

The detection engine answers:

"Is this something I should flag?"

2. Raw Log vs Security Event

A Linux log is just a text message.

For example:

2026-08-12T17:17:22.332098+00:00 ip-172-31-28-255 sshd-session[44055]:
Invalid user testuser123 from 152.57.19.200 port 46203

The detection engine turns this into something easier to work with:

Event Type : SSH Failed Login
Rule       : SSH-001
Severity   : 7
Username   : testuser123
Source IP  : 152.57.19.200
Port       : 46203
MITRE      : T1110

The original log message is still kept for investigation.

3. Current Detection Rules

The current implementation focuses on four main security activities.

Rule ID

Detection

Severity

MITRE

SSH-001

Failed SSH / Invalid User

7

T1110

SSH-002

Successful SSH Login

3

T1078

SUDO-001

Sudo Command Executed

4

T1548.003

CORR-SSH-001

Repeated SSH Failures

10

T1110

The severity values are used by LogRadar's dashboard and are not meant to represent an official universal SIEM scoring system.

4. SSH Failed Login

Example

A controlled SSH attempt against the VPS:

ssh baduser@<VPS-IP>

can produce:

Invalid user baduser from 152.57.18.51 port 1894

The parser extracts the useful fields:

event_type = ssh_failed
username   = baduser
source_ip  = 152.57.18.51
port       = 1894

The detection engine then applies:

Rule     : SSH-001
Severity : 7
MITRE    : T1110

Backend Output

A processed event can look like:

{
  "event_type": "ssh_failed",
  "username": "baduser",
  "source_ip": "152.57.18.51",
  "severity": 7,
  "rule_id": "SSH-001",
  "rule": "Failed SSH Login - Invalid User",
  "mitre": "T1110"
}

5. Successful SSH Login

Not every SSH event is an attack.

For example:

Accepted publickey for ubuntu from 152.57.19.200 port 59310 ssh2

is a successful authentication event.

LogRadar records it as:

Event    : Successful SSH Login
Rule     : SSH-002
Severity : 3
MITRE    : T1078

The event is useful because successful authentication can be important during an investigation.

For example, an analyst may want to check:

Who logged in?
From which IP?
When?
Which authentication method was used?

6. Sudo Activity

Sudo activity is useful because it indicates that a command was executed with elevated privileges.

Example:

sudo: ubuntu : COMMAND=/usr/bin/cat /etc/hosts

The detection engine can classify it as:

Event    : Sudo Command Executed
Rule     : SUDO-001
Severity : 4
MITRE    : T1548.003

The original command is preserved in the event so it can be reviewed later.

7. Rule Matching

The basic rule flow is:

Raw message
     ↓
Normalize message
     ↓
Check known patterns
     ↓
Matching rule found?
     ↓
YES ──→ Create security event
     │
     └──→ Assign severity + rule + MITRE

For example:

"Invalid user baduser from 152.57.18.51"

contains the pattern:

invalid user

so it can be classified as an SSH authentication failure.

8. Why Rule Order Matters

Detection rules are checked in a specific order.

For example, an SSH message containing:

invalid user

should be handled by the SSH failure rule before it falls through to a generic system event rule.

The general idea is:

Specific security rule
        ↓
Other security rules
        ↓
Generic system event

If no security rule matches, LogRadar can still keep the event as a normal system event instead of dropping it.

Example:

Rule     : SYS-001
Severity : 1

This allows the original event to remain available for investigation.

9. Severity

LogRadar currently uses a simple severity scale.

1  → Normal system information
3  → Successful authentication
4  → Sudo activity
7  → Failed SSH authentication
10 → Correlated brute-force alert

The purpose is to make important events easier to identify in the dashboard.

For example:

Severity 1
System Information

does not need the same attention as:

Severity 7
Failed SSH Login

10. MITRE ATT&CK Mapping

The detection rules also contain a MITRE ATT&CK reference.

For example:

Failed SSH
    ↓
T1110
    ↓
Brute Force

And successful authentication is associated with:

T1078
Valid Accounts

The MITRE field gives additional context to an alert.

It does not mean that every matching event proves that the corresponding technique was successfully used.

It is a classification used by the project to provide investigation context.

11. Detection Example

Suppose the agent receives:

[LOG] /var/log/auth.log:
Invalid user baduser from 152.57.18.51 port 1894

FastAPI receives:

POST /logs HTTP/1.1" 200

The parser extracts:

username   = baduser
source_ip  = 152.57.18.51
event_type = ssh_failed

The detection engine matches:

SSH-001

and produces:

Severity : 7
Rule     : Failed SSH Login - Invalid User
MITRE    : T1110

The event is then stored and becomes available to the React dashboard.

12. Detection vs Correlation

These are two different stages.

Detection

A single log entry is enough.

Invalid user baduser
        ↓
SSH-001
        ↓
Failed SSH Login

Correlation

Multiple events are required.

Failed SSH
Failed SSH
Failed SSH
Failed SSH
Failed SSH
        ↓
Same source IP
        ↓
Time window
        ↓
CORR-SSH-001
        ↓
Brute Force Alert

This distinction is important because a single failed login and repeated authentication attempts represent different levels of evidence.

13. Current Detection Pipeline

                    RAW LINUX LOG
                         │
                         ▼
                  ┌──────────────┐
                  │    Parser    │
                  └──────┬───────┘
                         │
                         ▼
                  Structured Event
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           SSH Rule   Sudo Rule   Other
              │          │          │
              └──────────┼──────────┘
                         ▼
                   Security Event
                         │
                         ▼
                    PostgreSQL
                         │
                         ▼
                  Correlation
                         │
                         ▼
                      Alert
                         │
                         ▼
                 React Dashboard

14. Current Scope

The detection engine is intentionally small.

It currently focuses on events that are useful for demonstrating a basic SIEM workflow:

SSH authentication failures

Successful SSH authentication

Sudo activity

Repeated SSH failures

The next stage of the project can expand the rules without changing the basic log collection pipeline.
