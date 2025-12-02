# Silva Analytics Platform - Overview

Analytics and customer intelligence platform for Silva Nutrition supplements network. Unifies data from WordPress e-commerce, Bling ERP, CRM, and AI models into single source of truth for real-time operational and marketing insights.

## Problem

Data scattered across multiple systems (WordPress, Bling ERP, CRM, AI models) prevents unified view of business performance. No real-time visibility into customer behavior, campaign effectiveness, or operational KPIs. Manual reporting is time-consuming and error-prone.

## Solution

Three-tier platform: FastAPI backend for data aggregation and analytics, Next.js dashboard for visualization, Supabase for data warehouse and RFM analysis. Integrates multiple data sources, provides 360° analytics dashboards, customer segmentation with RFM, and campaign monitoring.

## Key Features

- 360° analytics dashboards (e-commerce, CRM, marketing)
- Customer segmentation with RFM analysis
- Campaign and AI model monitoring
- Historical data snapshots with advanced filters
- Real-time KPI tracking
- Data export capabilities

## Stack

**Backend**: Python 3.11, FastAPI, Pydantic, SQLAlchemy, Supabase SDK  
**Frontend**: Next.js 15, TypeScript, Tailwind CSS, React Query, Recharts  
**Database**: Supabase (PostgreSQL) with RFM views  
**Monorepo**: Turborepo, PNPM workspaces

