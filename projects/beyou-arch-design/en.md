---
title: "Beyou Arch & Design"
summary: "Central repository for architecture and design documentation of the Beyou project, including diagrams, topic descriptions, and OpenAPI specifications."
---
# Beyou Arch & Design

This repository contains the architecture and design documentation for the Beyou project, serving as the single source of truth for system design decisions, API specifications, and user flows.

## Overview

The repository is organized into three main sections:

- **architecture** – high‑level system diagrams, component descriptions, and deployment topologies
- **design** – user‑interface mockups, interaction flows, and usability guidelines
- **api** – OpenAPI specifications for each endpoint group, along with controller‑level documentation

## Features

- Markdown‑based documentation with YAML metadata
- Multi‑language support (English and Portuguese)
- Diagram source files (Mermaid) that can be rendered as SVG
- Automated import into the Beyou backend for real‑time UI display
- Version‑controlled alongside code changes

## Usage

Developers and designers contribute to this repository by adding or updating markdown files in the appropriate folders. The documentation is automatically imported into the Beyou backend via the Docs Import API.

## Integration

The content is consumed by the Beyou Docs UI, which renders it in a user‑friendly interface. Changes pushed to the `main` branch trigger an import webhook that updates the live documentation.