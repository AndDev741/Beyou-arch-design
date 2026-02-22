---
title: "Beyou Dev Environment"
summary: "Development environment setup for the Beyou project, including Docker Compose configurations, database seeding, and local development tooling."
---
# Beyou Dev Environment

This repository provides a complete local development environment for the Beyou project, enabling developers to run the entire stack (backend, frontend, database) with a single command.

## Overview

The dev environment is built around Docker Compose and includes:

- **PostgreSQL** database with pre‑configured schemas
- **Backend** Spring Boot application with hot‑reload support
- **Frontend** React application served by Vite
- **Docs UI** running alongside the backend
- **Optional services** like MailHog for email testing

## Features

- One‑line startup: `docker‑compose up`
- Database seeding with realistic test data
- Environment‑variable management via `.env` files
- Integrated health checks and logging
- Support for multiple profiles (development, testing, CI)

## Usage

Clone the repository and run:

```bash
docker‑compose up -d
```

Access the services at:

- Backend API: `http://localhost:8080`
- Frontend: `http://localhost:5173`
- Docs UI: `http://localhost:3000`
- MailHog: `http://localhost:8025`

## Customization

The environment can be tailored by editing the `docker‑compose.yml` file and adjusting environment variables in the `.env` file. Additional services (e.g., Redis, Elasticsearch) can be added as needed.