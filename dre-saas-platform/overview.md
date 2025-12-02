# DRE Engine - Overview

Multi-tenant financial SaaS for DRE (Income Statement) generation, transaction management, and financial forecasting. Automated recurring transactions with smart versioning.

## Problem

Companies need automated financial statement generation and transaction management. Manual DRE creation is time-consuming. Recurring transactions require complex versioning when rules change.

## Solution

FastAPI backend with layered architecture (routers → services → repositories). Next.js dashboard for financial management. Supabase with RLS for multi-tenant isolation. Smart recurring rules with versioning support.

## Key Features

- DRE generation (monthly/yearly)
- Transaction management (income/expense)
- Recurring rules with smart versioning
- Financial forecasts
- Multi-tenant SaaS architecture
- CSV import

## Stack

**Backend**: Python 3.11, FastAPI, Pydantic, Supabase SDK  
**Frontend**: Next.js 14, TypeScript, Tailwind CSS  
**Database**: Supabase (PostgreSQL with RLS)  
**Monorepo**: PNPM workspaces

