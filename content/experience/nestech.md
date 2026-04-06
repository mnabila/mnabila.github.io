+++
draft = false
date = '2025-08-24T02:52:17+07:00'
title = 'CV. Nestech'
type = 'experience'
role = 'Backend Engineer'
period = 'April 2022 - July 2022'
period_begin = '2022-04-07'
period_finish = '2022-07-14'
location = 'South Jakarta, Jakarta'
workmode = 'remote'
worktype = 'freelance'
technologies= [
   'docker',
   'golang',
   'nginx' ,
   'postgresql',
]
tasks = [
  'Built membership management service for provincial and district levels',
  'Integrated OCR for automated multi-location data input',
  'Optimized data processing for faster membership updates',
]
+++

## Overview

Worked as a freelance Backend Engineer at CV. Nestech on a 3-month remote engagement, building a membership management system in Go with PostgreSQL. The project digitized membership data collection for an organization spanning provincial and district levels, replacing a manual paper-based process with a centralized backend service.

## Key Projects

### Membership Management Service

Built a membership management service in Go with PostgreSQL that handles member registration, data collection, and hierarchical organization across provincial and district levels. Designed the database schema to model the administrative hierarchy, enabling queries like "all members in district X under province Y" with efficient indexed lookups. Exposed RESTful APIs for CRUD operations, bulk imports, and reporting. Deployed with Docker and Nginx as a reverse proxy.

### OCR Data Input Automation

Integrated an OCR engine to automate data entry from physical membership forms submitted by district offices. The pipeline accepts scanned document images, extracts text fields (name, ID number, address), validates them against expected formats and PostgreSQL constraints, and imports valid records into the database. Invalid entries are flagged for manual review rather than silently dropped, ensuring data completeness.

### Data Processing Optimization

Optimized PostgreSQL queries and batch processing workflows to handle concurrent data submissions from multiple districts. Added database indexes on frequently queried columns, rewrote N+1 query patterns into batch queries, and implemented connection pooling to handle spikes when multiple districts submit data simultaneously.

## Impact

- Replaced a fragmented paper-based membership process with a centralized digital system accessible across all administrative levels
- Reduced manual data entry workload through OCR automation while maintaining data accuracy with validation and review workflows
- Improved query performance for membership reports, enabling administrators to access up-to-date data without delays
