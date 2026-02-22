---
title: "Beyou Backend"
summary: "Spring Boot backend for the Beyou application, providing REST APIs for goals, habits, tasks, and user management."
---
# Beyou Backend
Spring Boot backend for the Beyou application, providing REST APIs for goals, habits, tasks, and user management.

## Overview
This backend is built with Java 17, Spring Boot 3, PostgreSQL, and Docker. It follows a clean architecture with separate layers for controllers, services, and repositories.

## History
- **2024‑Q4**: Initial project setup with basic user authentication and goal tracking.
- **2025‑Q1**: Added habit tracking, routine scheduling, and category management.
- **2025‑Q2**: Implemented documentation import system for API, architecture, and design docs.
- **2025‑Q3**: Added Xp calculation, refresh tokens, and email notifications.

## Current Status
The backend is in active development with regular updates. It serves as the core of the Beyou productivity ecosystem.

## Architecture Highlights
- JWT‑based authentication with refresh tokens
- Role‑based authorization (USER, ADMIN)
- Database migrations with Flyway
- Integration tests with Testcontainers
- Centralized error handling and logging

## Deployment
- Docker container deployed to a private VPS
- CI/CD via GitHub Actions
- Environment‑specific configuration profiles