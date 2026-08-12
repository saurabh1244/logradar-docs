# LogRadar

A lightweight Linux Security Monitoring and Mini-SIEM platform.

## Overview

LogRadar continuously collects Linux authentication and system logs,
analyzes security events, detects suspicious activity, and displays
security telemetry through a React dashboard.

## Architecture

VPS → LogRadar Agent → FastAPI → Detection Engine
→ PostgreSQL → React Dashboard

## Features

- Real-time Linux log monitoring
- SSH authentication detection
- Sudo activity detection
- SSH brute-force correlation
- MITRE ATT&CK mapping
- Security severity scoring
- Persistent PostgreSQL storage
- React SIEM dashboard
- Alert investigation
- Search and filtering

## Tech Stack

- Python
- FastAPI
- React
- PostgreSQL / Neon
- Linux
- Docker

## Documentation

- [Architecture](docs/architecture.md)
- [Detection Rules](docs/detection-rules.md)
- [Setup Guide](docs/setup.md)
- [Attack Simulation](docs/attack-simulation.md)
- [API Documentation](docs/api.md)
