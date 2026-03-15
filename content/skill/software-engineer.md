+++
draft = false
date = '2025-08-24T11:54:13+07:00'
title = 'Software Engineer'
type = 'skill'
description = 'Building scalable and maintainable software solutions using modern programming languages, frameworks, and best practices.'
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

My primary focus is backend development — designing APIs, building services, and writing the kind of code that holds up under real traffic and real requirements. I work primarily in Go and TypeScript, with SQL as the glue that connects most of it to persistent storage.

## What I Do

**Backend services and APIs.** Most of my work involves building HTTP services that other systems consume. I design clean API contracts (typically REST with OpenAPI specs), implement layered architectures that separate routing from business logic, and make sure the service handles failure gracefully. I have built session-managed automation servers, workflow APIs, and integration layers connecting internal tools to external platforms.

**Database design and optimization.** I work with PostgreSQL and MySQL/MariaDB for relational data, Redis for caching and ephemeral state, and RabbitMQ for asynchronous messaging. I write raw SQL when the ORM gets in the way and design schemas that reflect the actual domain rather than the framework's defaults.

**Frontend when the project needs it.** I am not a frontend specialist, but I can build functional interfaces with Next.js, Vite, and Tailwind CSS when the project calls for it. I reach for Ant Design when I need a component library that handles the common patterns without reinventing them.

## How I Work

I value code that is easy to read, easy to change, and easy to delete. I use layered architectures — controllers, services, data access — not because they are trendy, but because they make it possible to change one part of the system without breaking everything else.

I lean on typed languages and validation at system boundaries. In Go, I use the type system to make invalid states unrepresentable. In TypeScript, I use Zod for runtime validation and let the types propagate from there. Inside the application, I trust the code — no redundant nil checks, no defensive copies where they are not needed.

Version control is part of the workflow, not an afterthought. I use Git with clear commit history, meaningful branch names, and pull requests that can be reviewed without a walkthrough call.
