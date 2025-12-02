# FAMATH Enrollment System - Overview

Web-based enrollment management system for FAMATH. Complete student enrollment workflow from registration to completion, with Kanban visualization, voucher management, and commercial analytics.

## Problem

Manual enrollment process lacks visibility and control. No centralized system for tracking candidates, managing documents, or monitoring commercial performance. Voucher and campaign management requires manual coordination.

## Solution

Layered architecture: React frontend with Kanban board, Express backend API, Supabase database. Department-based permissions, voucher system with campaigns, commercial dashboard with analytics.

## Key Features

- Enrollment Kanban (drag & drop)
- Voucher and campaign management
- Document management
- Commercial dashboard with analytics
- Department-based permissions
- Responsible assignment system

## Stack

**Frontend**: React 18, TypeScript, Vite, Tailwind CSS  
**Backend**: Node.js, Express, TypeScript  
**Database**: Supabase (PostgreSQL with RLS)  
**Monorepo**: PNPM workspaces

