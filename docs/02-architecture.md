# Architecture

## High-Level Flow

The current architecture is:

Linux VPS
↓
LogRadar Agent
↓
FastAPI `/logs`
↓
Log Parser
↓
Detection Rules
↓
Event Storage
↓
Correlation Engine
↓
Alerts
↓
React Dashboard

## 1. Linux VPS

The VPS is the monitored machine.

Linux generates logs as different system and authentication activities happen.

For example, an SSH attempt can generate entries in:

`/var/log/auth.log`

System activities are commonly available through:

`/var/log/syslog`

## 2. LogRadar Agent

The Python agent runs on the VPS.

Its job is simple:

1. Watch the configured log files.
2. Detect newly added lines.
3. Read the new line.
4. Send it to the FastAPI backend.

The agent does not decide whether an event is malicious.

It mainly acts as the collector.

## 3. FastAPI Backend

FastAPI receives the log entry through:

`POST /logs`

The request contains:

- hostname
- source log file
- raw log message

The backend then processes the message.

## 4. Parser

The parser looks at the raw log message and extracts useful information.

For example:

```text
Invalid user testuser123 from 152.57.19.200 port 46203
