# KukkuOne — Documentation

**KukkuOne** is a B2B Poultry Management SaaS ERP connecting **Suppliers → Farmers → Distributors** through the full commercial + production loop (procurement → placement → daily farming → harvest → purchase → settlement → next batch).

The **Farmer ERP is the central engine** and is built **first**. Supplier and Distributor modules plug in on the same architecture afterward.

This folder is the **single source of truth** for building KukkuOne. Read the documents in order the first time; use them as reference thereafter.

---

## How to use these docs

1. Start with **[00-Decision-Record](00-Decision-Record.md)** — the locked tech + architecture decisions and *why*. Everything else assumes these.
2. Read **[01-Product-Overview](01-Product-Overview.md)** for the domain, actors, and MVP scope.
3. Read **[02-System-Architecture](02-System-Architecture.md)** and **[04-Repository-Structure](04-Repository-Structure.md)** to understand the shape of the codebase.
4. When building a feature, open its spec doc (Farmer / Admin / Mobile) plus **[06-Data-Model](06-Data-Model.md)** and **[12-API-Conventions](12-API-Conventions.md)**.
5. Follow **[13-Roadmap-And-Milestones](13-Roadmap-And-Milestones.md)** for the phased build order.

---

## Document index

| # | Document | What it covers |
|---|----------|----------------|
| 00 | [Decision Record](00-Decision-Record.md) | Locked tech stack + architecture decisions, evaluation of your suggestions, rationale |
| 01 | [Product Overview](01-Product-Overview.md) | Vision, actors, MVP scope, end-to-end business flow, acceptance criteria |
| 02 | [System Architecture](02-System-Architecture.md) | Gateway + service APIs, data flow, diagrams, scaling path |
| 03 | [Tech Stack](03-Tech-Stack.md) | Every technology chosen, versions, rationale, alternatives considered |
| 04 | [Repository Structure](04-Repository-Structure.md) | Monorepo layout, all packages/apps, shared code, naming |
| 05 | [Auth & RBAC](05-Auth-And-RBAC.md) | Pluggable auth layer, Firebase provider, swap strategy, roles/permissions, admin login |
| 06 | [Data Model](06-Data-Model.md) | Entities, ERD, Prisma schema sketch, per-actor extension points |
| 07 | [Farmer Module Spec](07-Farmer-Module-Spec.md) | **Build-first module**: farms, sheds, batches, Day 0, daily logs, feed, health, growth, finance, harvest — endpoints + screens |
| 08 | [Admin Panel Spec](08-Admin-Panel-Spec.md) | Sidebar/header/filter layout, pages, data views for the owner |
| 09 | [Client Apps Spec (Web + Mobile)](09-Client-Apps-Spec.md) | One role-aware **web** app + one Expo **mobile** app, signup capability selection, **how farmer/distributor/supplier differ**, farmer screens |
| 10 | [Environment & Config](10-Environment-And-Config.md) | Env strategy, one-connection philosophy, `.env.example` per app, secrets |
| 11 | [Local Setup & Deploy](11-Local-Setup-And-Deploy.md) | Run everything locally (Docker Compose), host-anywhere deploy |
| 12 | [API Conventions](12-API-Conventions.md) | REST conventions, gateway routing, errors, pagination, versioning |
| 13 | [Roadmap & Milestones](13-Roadmap-And-Milestones.md) | Phased plan, farmer-first, milestones mapped to acceptance criteria |

---

## The one-paragraph summary

Build a **pnpm + Turborepo monorepo**. A **NestJS API Gateway** verifies auth and routes to three **NestJS service APIs** (`farmer-api`, `supplier-api`, `distributor-api`) sharing one **PostgreSQL** database (via **Prisma**, one `DATABASE_URL`). Auth is a **pluggable layer** defaulting to **Firebase Google Authentication**, swappable via a single adapter. There are **two React (Vite + TypeScript) web apps** — one **role-aware actor app** (`kukkuone-web`) and a separate **owner console** (`kukkuone-admin-web`) — plus **one Expo/React Native mobile app**. At signup the user selects the org's **capabilities** (Farmer / Distributor / Supplier — any or all); both web and mobile then render the matching experience from those **capabilities + role**. Everything is **env-driven** so the same build runs locally (Docker Compose) or on any host with one connection string. **Farmer module ships first.**
