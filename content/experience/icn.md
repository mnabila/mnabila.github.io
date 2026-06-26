+++
draft = false
date = '2025-08-24T02:53:02+07:00'
title = 'PT. Insan Cendekia Nusantara'
type = 'experience'
role = 'Backend Engineer'
period_begin = '2022-06-01'
period_finish = '2023-06-30'
location = 'Tulungagung, East Java'
workmode = 'onsite'
worktype = 'full-time'
technologies= [
   'docker',
   'golang',
   'javascript',
   'postgresql',
]
tasks = [
  'Built PPOB backend with multi-provider payment integration',
  'Developed Epita e-learning platform for content delivery',
]
impact = [
  'Enabled reliable multi-provider payment processing with consistent transaction tracking, reducing failed payment issues through retry mechanisms',
  'Delivered a functional e-learning platform that allowed instructors to publish courses and learners to track progress, from initial design to production deployment',
]
+++

## Overview

Built two products from scratch for a company entering the digital payment and edtech space. The first was a PPOB payment service integrating multiple third-party providers with different API contracts into a single unified backend. The second was Epita, an e-learning platform with course management and progress tracking. Both projects went from initial API design through to production deployment, giving me ownership across the full development lifecycle.

## Key Projects

### PPOB Backend Service

Built the backend for a PPOB (Payment Point Online Bank) service in Go, enabling users to pay electricity bills, purchase mobile top-ups, and buy digital products through a unified platform. Integrated multiple payment providers (each with different API contracts and authentication methods) behind a consistent internal API. Implemented transaction state management with SQL to track payment lifecycle from initiation through provider confirmation to settlement, with retry logic for handling provider timeouts.

### Epita E-Learning Platform

Developed the backend for Epita, an e-learning platform built with Go and JavaScript. Designed RESTful APIs for course management, content uploads, user enrollment, and progress tracking. Instructors can create and organize course materials, while learners access content with their progress synced across sessions. Used Nginx to serve static assets and handle SSL termination.
