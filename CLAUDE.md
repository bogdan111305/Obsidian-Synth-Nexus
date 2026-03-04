# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

An Obsidian Personal Knowledge Management (PKM) vault focused on Backend Development and Microservices Architecture. It contains 219+ Markdown notes covering Java, Spring, Hibernate, JDBC, Kafka, Git, and observability tooling — organized as a learning reference library. There are no build scripts, test runners, or executables.

## Vault Layout

- `Base/` — All knowledge notes, organized by technology domain
  - `Java/`, `Spring/`, `Hibernate/`, `JDBC/`, `Kafka/`, `Git/`, `Migration/`, `Grafana, Prometheus, OpenTelemetry и Jaeger/`
  - `Проекты/` — Project design documents (in Russian)
- `Excalidraw/` — Architectural diagrams as `.excalidraw.md` files
- `Schemas/` — Canvas-based diagrams (Obsidian canvas format)
- `Files/` — Images and attachments referenced by notes (never edit directly)
- `.obsidian/` — Obsidian configuration (plugins, hotkeys, appearance)

New notes created through Obsidian are placed in `Base/` by default. Attachments go to `Files/`.

## Obsidian Configuration

Community plugins in use: `obsidian-excalidraw-plugin`, `obsidian-git`, `cursor-bridge`. Settings are global (no per-workspace overrides). The gitignore excludes `.obsidian/workspace.json` and similar session noise.

## Git Workflow

Commits are made via `obsidian-git` plugin or manually. Commit style is either:
- `vault backup: YYYY-MM-DD HH:MM:SS` — automated backups
- `<Category>: <description>` — intentional changes (e.g., `UI: ...`, `Cleanup: ...`)

## Main Project Being Documented

The `Base/Проекты/` directory contains a detailed microservices design for a **Corporate Social Network** with 10 services:

| Service | Stack |
|---|---|
| User Management | Java/Spring Boot + MySQL/PostgreSQL |
| Authentication | Java/Spring Security + OAuth2/OIDC/SAML (Keycloak) |
| Chat | Scala/Akka HTTP + MongoDB (WebSocket) |
| Video Conferencing | Java + WebRTC/Agora |
| Document Service | Java/Spring + S3/DocuSign |
| Notification | Scala/Akka Streams + Kafka |
| Search | Scala/Play + Elasticsearch |
| API Gateway | Java/Spring Cloud Gateway |
| Service Registry | Java/Eureka |
| Monitoring | Java/Spring Boot Admin + Prometheus/Grafana/Jaeger |

Communication: REST, WebSocket (real-time), Kafka (event-driven). Frontend: Flutter.

## Note Conventions

- Notes are written in Russian or English (mixed throughout the vault)
- Excalidraw diagrams use the `.excalidraw.md` extension and are embedded in regular notes via `![[filename.excalidraw]]`
- Canvas files (`.canvas`) are JSON-based Obsidian canvases linking related notes visually
- Internal links use Obsidian wikilink syntax: `[[Note Title]]`
