# LogRadar — Project Overview

## What is LogRadar?

LogRadar is a small Linux security monitoring system that I built to understand how a basic SIEM pipeline works.

The system monitors security-related logs from a Linux VPS and sends new log entries to a FastAPI backend.

The backend analyzes the logs, identifies known security events, stores them in PostgreSQL, and exposes the results through APIs.

A React dashboard is used to view the events and alerts.

## Main Goal

The main goal of the project is to understand the complete flow:

Linux log → Collection → API → Parsing → Detection → Correlation → Database → Dashboard

I wanted to build the flow myself instead of only using an existing SIEM product.

## What it currently monitors

The agent currently watches:

- `/var/log/auth.log`
- `/var/log/syslog`

The authentication log is especially useful for SSH activity, user sessions, sudo activity and other authentication-related events.

## Current detections

The detection layer currently handles events such as:

- Failed SSH login
- Invalid SSH user
- Successful SSH login
- Sudo command execution
- Repeated SSH failures from the same IP

Repeated SSH failures are handled differently from a single failed login.

A single failed login is an event, while multiple failures from the same source IP within a short time window can become a security alert.

## Technology

- Python
- FastAPI
- React
- PostgreSQL
- Linux
- Neon PostgreSQL
- Tailwind CSS

## Project Status

This is a personal learning and portfolio project.

It is intentionally kept small so that I can understand each part of the monitoring pipeline instead of hiding the implementation behind a large security platform.
