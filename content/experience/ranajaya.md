+++
draft = false
date = '2025-08-24T02:52:31+07:00'
title = 'PT. Ranajaya Citaprasada Thani Indonesia'
type = 'experience'
role = 'Backend Engineer'
period_begin = '2023-07-01'
period_finish = ''
location = 'Depok, West Java'
workmode = 'onsite'
worktype = 'full-time'
technologies= [
  'docker',
  'golang',
  'javascript',
  'minio',
  'rabbitmq',
  'redis',
  'mysql',
  'postgresql',
  'typescript',
  'appium',
  'playwright',
  'reactjs',
  'vitejs',
]
tasks = [
  'Built GrabPay payment gateway aggregator with multi-provider integration and smart routing system',
  'Created CLI tool for automated log collection and API serving',
  'Built data importer for SMS blasting system',
  'Developed core backend for EDC platform',
  'Led backend team with code reviews and technical guidance',
  'Mentored junior developers and improved team productivity',
  'Managed Docker images and standardized deployment workflows',
  'Built loan mini apps dashboard for monitoring loan transactions',
  'Managed Rubee Dashboard backend with cross-team collaboration',
]
+++

## Overview

Working as a Backend Engineer at PT. Ranajaya Citaprasada Thani Indonesia, building backend systems across payment gateways, internal tooling, and electronic transaction platforms using Go, TypeScript, and various infrastructure tools. Taking on leadership responsibilities by leading the backend team, conducting code reviews, and mentoring junior developers.

## Key Projects

### GrabPay Payment Gateway Aggregator

Built a payment gateway aggregator service in Go with multi-provider integration, unifying several third-party payment providers behind a single API for GrabPay transactions. Designed a smart routing system that selects the optimal provider per transaction based on success rate, latency, and provider health scores tracked in Redis. The routing engine supports automatic failover: when a primary provider fails or responds above the latency threshold, the system reroutes through alternative providers without interrupting the user flow. Implemented provider-level circuit breakers to prevent cascading failures across the payment pipeline. Used RabbitMQ for asynchronous reconciliation jobs that match settlement records across providers and flag discrepancies for manual review.

### CLI Log Collection Tool

Built a CLI tool in Go that connects to multiple service instances, collects and aggregates log data, and serves it through a REST API. The tool parses logs into structured JSON format, supports filtering by service name, severity, and time range, making it easier for the team to debug production issues without SSH-ing into individual servers.

### SMS Blasting Data Importer

Developed a data importer in Go that processes bulk recipient data for the SMS blasting system. The importer reads large CSV/Excel files, validates phone numbers and message templates, deduplicates entries, and pushes clean data into RabbitMQ queues for the SMS gateway to consume. Handles batch processing to avoid memory spikes on large datasets.

### EDC Platform Core Backend

Developed and maintain the core backend for an EDC (Electronic Data Capture) platform in Go. The system manages the full transaction lifecycle, from device registration and terminal configuration to transaction processing and end-of-day settlement. Handles communication with physical EDC terminals over TCP, processing transaction requests and returning authorization responses.

### Loan Mini Apps Dashboard

Built a monitoring dashboard for loan mini apps that provides real-time visibility into loan transaction activity. The backend is built in Go with MySQL, serving a React frontend bootstrapped with Vite. Exposes APIs for tracking transaction status, loan disbursement progress, and repayment records. Implemented filterable views by transaction state, date range, and partner channel, along with summary endpoints for aggregate metrics like approval rates and outstanding balances. Designed the API layer to support the frontend's need for paginated transaction lists and exportable reports.

### Rubee Dashboard

Managed the backend for "Rubee Dashboard", a centralized management platform built with Go and PostgreSQL. Designed RESTful APIs consumed by the React frontend, implemented role-based access control, and built reporting endpoints with complex SQL aggregations. Collaborated closely with UI/UX, frontend, and QA teams to align API contracts and deliver features iteratively.

### Docker Infrastructure

Standardized deployment across all services by creating optimized multi-stage Docker builds for Go applications, configuring Nginx as a reverse proxy, and setting up MinIO for object storage. Established Docker Compose configurations for local development that mirror production topology, reducing "works on my machine" issues.

## Impact

- Reduced payment transaction failures through smart routing system that selects optimal providers based on success rate, latency, and health scores, with circuit breakers and automatic failover ensuring uninterrupted payment flow
- Enabled real-time loan transaction monitoring with filterable views and aggregate metrics, replacing manual database queries with a self-service dashboard for operations and partner teams
- Improved debugging efficiency by replacing manual SSH log inspection with a centralized CLI tool and API
- Enabled the backend team to ship higher-quality code through structured code reviews and established Go coding standards
- Cut junior developer onboarding time through hands-on mentorship and pair programming sessions
- Eliminated environment drift between development and production by standardizing Docker-based deployment workflows
