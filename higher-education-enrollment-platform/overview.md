# Overview

The Higher Education Enrollment Platform manages the student enrollment lifecycle for a higher education institution: candidate registration, course selection, document review, pricing rules, payment workflow, and internal backoffice operations.

The project started as a rebuild after a technical audit found that the legacy system mixed browser-side business rules, direct database access from frontend code, weak permission boundaries, exposed credentials, and large client-side data loads. The safest path was not to keep patching the old codebase. The new platform was built with a backend-centered architecture and migrated gradually to production while the legacy system remained available during user adaptation.

## What Changed

**Before**: frontend code could bypass backend protections, sensitive workflows depended on browser-side logic, operational screens loaded too much data client-side, and permissions were difficult to trust.

**After**: core writes and business rules go through a backend API; authentication and authorization are checked server-side; pricing and payment workflows are centralized; relevant operational actions are logged; and the database boundary is treated as a defense-in-depth layer rather than the primary application interface.

## Platform Scope

### Student Portal

Candidates register, choose courses and shifts, submit required documents, follow enrollment status, and complete payment through a controlled server-side flow.

### Internal Backoffice

Internal users manage enrollments, documents, vouchers, campaigns, collaborators, and operational stages through role-aware workflows. Sensitive actions are protected by backend permission checks and recorded in audit logs.

### Backend API

The API is the main system boundary. It owns authentication, RBAC, pricing rules, payment orchestration, persistence, document operations, and audit logging.

### Data and Migration

The database schema was normalized around enrollment, documents, collaborators, pricing, campaigns, vouchers, payments, sessions, and audit logs. The migration strategy kept the legacy system available while the new system entered production, reducing cutover risk and giving users time to adapt.

## Engineering Focus

- Move critical business rules from browser to backend.
- Keep payment credentials and database credentials server-side.
- Enforce permissions in the API, not in editable frontend state.
- Add audit logs for collaborator actions and sensitive workflow changes.
- Use pagination and backend queries for operational screens.
- Preserve production continuity during migration.
- Keep architecture understandable for a small team to maintain.

## Evidence

- `architecture.md` — system architecture and boundaries.
- `security-and-data-boundaries.md` — auth, RBAC, secrets, and sensitive data handling.
- `key-flows.md` — enrollment, document, pricing, payment, and backoffice flows.
- `technical-decisions.md` — trade-offs and decisions.
- `results-and-measurement.md` — outcomes and next measurements.
