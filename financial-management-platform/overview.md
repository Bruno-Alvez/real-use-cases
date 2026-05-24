# Overview

## Context

Brazilian small and mid-size companies manage their finances in spreadsheets long after it stops being viable. The typical pattern is a shared Excel file with manual DRE consolidation at month-end, recurring transactions re-entered by hand each cycle, and category hierarchies that drift over time as different people touch the same file. When a company grows past a handful of accounts, that process breaks down, but the tooling doesn't change because no one has built the right thing for the Brazilian market at an accessible price point.

The project was built to fill that gap: a multi-tenant SaaS platform where companies manage their financial operations end-to-end, with tenant isolation enforced at the database layer so no company's data is ever reachable by another, regardless of what the application code does.

## Problem

Companies using spreadsheet-based financial management face three recurring failure modes. First, DRE generation is a manual consolidation exercise that consumes hours of work each month and is error-prone by nature. Second, recurring transactions (rent, subscriptions, payroll categories) are re-entered or copied manually every cycle, creating duplicates and gaps when someone forgets. Third, there is no reliable forecast: companies project revenue and expenses by eyeballing historical averages, with no mechanism to account for trend or seasonality.

Layered underneath those operational problems is a structural one. A SaaS platform serving multiple companies cannot trust application code alone to enforce data isolation. A query that forgets a `WHERE company_id = ?` clause will return another tenant's data. That kind of bug needs to be caught at the database layer, not in a code review.

## Solution

The platform delivers four capabilities, all built on a FastAPI backend with a strict router-to-service-to-repository layering, a Next.js dashboard, and PostgreSQL via PostgreSQL with Row Level Security as the tenant isolation boundary.

DRE generation is automated: the system aggregates transactions by income and expense categories across any time range and produces a structured monthly or annual statement, replacing the spreadsheet consolidation process entirely.

Recurring rule processing handles the lifecycle of repeating transactions with a versioning model that separates "update all past and future occurrences" from "update future only." The latter creates a new rule version linked to the original, so historical transactions remain accurate while future ones reflect the change.

Transaction deduplication ensures that recurring rules can be processed multiple times safely. A composite key over date, amount, description, category, and account uniquely identifies each generated transaction, so reprocessing produces the same result rather than duplicates.

The forecasting engine provides two modes. The basic mode uses historical averages. The Pro plan mode applies EWMA smoothing, trend detection via linear regression over smoothed values, and seasonality adjustment from historical month-over-month patterns. Results are stored with their algorithm metadata, making it possible to audit how any forecast was produced.

## Key Features

- Automated DRE generation, monthly and annual, in under 30 seconds
- Transaction management with income and expense categorization
- Recurring rules with smart versioning for "update future only" use cases
- Financial forecasting with EWMA, trend detection, and seasonality (Pro plan)
- Full multi-tenant isolation via RLS, enforced at PostgreSQL layer
- CSV import with partial success model and per-import history tracking
- Plan-based feature gating (Simple vs Pro) enforced at the service layer

## Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11, FastAPI 0.104+, Pydantic 2.5+ |
| Database | PostgreSQL via PostgreSQL, Row Level Security |
| Data processing | pandas (CSV import), reportlab (PDF generation) |
| Auth | JWT validation, python-jose, company_id via database lookup |
| HTTP client | httpx (private object storage operations) |
| Frontend | Next.js, TypeScript |
| Infrastructure | Docker, Docker Compose, Uvicorn ASGI |
