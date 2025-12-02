# FAMATH Enrollment Platform - Overview

Complete enrollment management platform for FAMATH. End-to-end student enrollment workflow from registration to payment, with document management, smart pricing, and Asaas payment integration.

## Problem

Manual enrollment process lacks automation. No integrated payment system. Document management is fragmented. Pricing calculation is complex and error-prone.

## Solution

Monorepo platform: React frontend for student portal, Express backend API, Supabase for data and storage, Asaas for payments. Smart pricing engine, document upload system, checkout flow.

## Key Features

- Student enrollment portal
- Smart pricing calculation (date-based)
- Document upload with Supabase Storage
- Asaas payment integration (PIX, Credit Card)
- Voucher system
- Enrollment tracking timeline
- Checkout session management

## Stack

**Frontend**: React 18, TypeScript, Vite, Tailwind CSS  
**Backend**: Node.js, Express, TypeScript  
**Database**: Supabase (PostgreSQL + Storage)  
**Payments**: Asaas API  
**Monorepo**: PNPM workspaces

