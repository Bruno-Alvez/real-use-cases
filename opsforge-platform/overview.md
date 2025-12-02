# OpsForge - Overview

SaaS observability platform for integrations, infrastructure, and webhook delivery. Multi-tenant system providing monitoring, alerting, and delivery guarantees for API integrations.

## Problem

Teams building integrations need observability into webhook delivery, endpoint health, and infrastructure metrics. Building observability tooling in-house is time-consuming and diverts focus from core product. Existing solutions are either too generic or too expensive.

## Solution

SaaS platform with three core services: API (Go + Gin) for ingestion and management, Worker (Go) for background processing and monitoring, Dashboard (Next.js) for visualization. Multi-tenant architecture with PostgreSQL RLS. Prometheus + OpenTelemetry for metrics collection. Designed to scale from startups to enterprises.

## Key Features

- Webhook delivery with automatic retries
- Endpoint health monitoring and uptime tracking
- Real-time metrics and dashboards
- Multi-tenant SaaS architecture
- Prometheus metrics export
- Infrastructure monitoring (planned)
- Alerting and incident management (planned)
- SLO tracking (planned)

## Stack

**Backend**: Go 1.21, Gin, pgx, zap  
**Worker**: Go background jobs, ticker-based polling  
**Frontend**: Next.js 14, TypeScript, Tailwind, shadcn/ui  
**Database**: PostgreSQL (Supabase compatible) with RLS  
**Observability**: Prometheus, OpenTelemetry, Grafana  
**Monorepo**: Turborepo

