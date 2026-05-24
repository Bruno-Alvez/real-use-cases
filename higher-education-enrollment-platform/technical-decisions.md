# Technical Decisions

## API as the only trusted application boundary

**Context:** The legacy system allowed frontend code to perform sensitive operations directly against the database. That made backend validations, permissions, and logging easy to bypass.

**Decision:** The new system treats the backend API as the boundary for state changes. Frontends call the API; the API validates input, checks authentication and permissions, executes writes, and records audit events.

**Trade-off:** This adds backend work and requires clear API contracts, but it makes security, validation, and observability enforceable in one place.

## Server-side authentication and RBAC

**Context:** Internal users have different responsibilities: document review, enrollment operations, campaign/voucher management, and administrative functions.

**Decision:** Authentication and authorization are enforced server-side with JWT-based sessions, role/permission checks, and backend guards around sensitive routes.

**Trade-off:** Permission modeling takes more work upfront, but it avoids trusting editable browser state for access control.

## Pricing calculated on the backend

**Context:** Pricing depends on course, shift, campaigns, vouchers, deadlines, and payment rules. In the legacy system, part of this logic lived in browser code.

**Decision:** Pricing rules are centralized in backend/domain code and persisted in canonical pricing tables.

**Trade-off:** The frontend becomes less autonomous, but pricing is no longer manipulable by changing client-side state.

## Audit trail for operational actions

**Context:** Document approval, enrollment stage changes, voucher usage, payment confirmation, and collaborator actions are sensitive operational events.

**Decision:** Relevant actions are written to append-style audit logs with actor, action, entity, timestamp, and metadata.

**Trade-off:** This adds write volume and requires discipline in services, but gives the organization traceability for support, governance, and incident analysis.

## Gradual migration instead of hard cutover

**Context:** Replacing a live enrollment system can interrupt users and internal teams if done as a single big-bang release.

**Decision:** The new platform entered production while the legacy system remained available during user adaptation. Cutover risk was reduced by keeping rollback and comparison paths open.

**Trade-off:** Running old and new systems in parallel is operationally heavier, but it reduces the probability of blocking enrollment operations.

## Cloud deployment with AWS operational patterns

**Context:** The system needed a practical production setup under deadline pressure while still keeping logs, secrets, and operational visibility under control.

**Decision:** Deployment and operations followed cloud-first patterns: containerized backend, managed frontend delivery, environment-scoped secrets, structured logs, and AWS observability/infrastructure practices where applicable.

**Trade-off:** The first production version optimized for delivery speed; the architecture keeps room for deeper AWS standardization as the platform stabilizes.
