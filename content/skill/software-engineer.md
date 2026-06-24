+++
draft = false
date = '2025-08-24T11:54:13+07:00'
title = 'Software Engineer'
type = 'skill'
description = 'Backend-focused development in Go and TypeScript and the occasional frontend when the project needs it.'
languages = ['golang', 'javascript', 'typescript', 'sql', 'rust']
tools = [
  'git',
  'github',
  'gitlab',
  'docker',
  'node.js',
  'postgresql',
  'mysql',
  'redis',
  'rabbitmq',
  "nextjs",
  "vitejs",
  "tailwindcss",
  "ant design",
]
+++

Most of my day-to-day is writing Go and TypeScript for backend services. SQL ties most of it together on the storage side.

## What I Do

**Backend services and APIs.** I build REST APIs, usually with an OpenAPI spec upfront so the contract is clear before I write any code. The services are structured as controllers, services, and data access layers. I've built payment gateway aggregators, chatbot automation servers, data importers, and dashboard backends. Different domains, but the workflow is similar: nail down the API contract, wire up the business logic, handle the edge cases.

**Databases and messaging.** PostgreSQL and MySQL/MariaDB for relational data, Redis for caching, RabbitMQ when I need async processing. I write raw SQL when the ORM fights me. I spend a fair amount of time on schema design, getting the data model right early saves a lot of pain later.

**Frontend when needed.** Not my main thing, but I can put together a working UI with Next.js, Vite, and Tailwind CSS. I use Ant Design when I need form components and tables without building them from scratch.

## How I Work

I split code into layers (controllers, services, data access) so I can change one part without touching everything else. It sounds obvious, but it matters when you're maintaining several services at once.

In Go, I lean on the type system to catch problems at compile time. In TypeScript, I use Zod to validate at the API boundary and trust the types from there with no extra nil checks deep inside the app where the data is already known to be valid.

I keep commit history clean, use meaningful branch names, and write PRs that someone can review by reading the diff.
