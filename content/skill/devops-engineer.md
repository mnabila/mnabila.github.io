+++
draft = false
date = '2025-08-24T11:54:19+07:00'
title = 'DevOps Engineer'
type = 'skill'
description = 'Designing and implementing CI/CD pipelines to automate software builds, tests, and deployments for efficient and reliable delivery.'
languages = ['bash', 'golang', 'python', 'rust']
tools= [
  'linux system administrator',
  'gitlab ci/cd', 
  'github actions', 
  'digitalocean', 'docker',
  'podman',
  'git'
]
+++

I treat infrastructure the same way I treat application code: it should be version-controlled, reproducible, and easy to reason about. My DevOps work focuses on CI/CD pipelines, containerization, and Linux server management — making sure code gets from a developer's machine to production reliably and without manual intervention.

## What I Do

**CI/CD pipeline design.** I build automated pipelines using GitLab CI/CD and GitHub Actions that handle the full lifecycle: build, test, lint, containerize, and deploy. I design pipelines that fail fast with clear error messages, cache aggressively to keep build times short, and only deploy when every check passes.

**Containerization and orchestration.** I use Docker and Podman to containerize applications with multi-stage builds that keep production images lean. I design Docker Compose configurations for local development environments — standardized infrastructure stacks that any team member can spin up with a single command. I have built custom container images when the official ones do not include the plugins or configurations a project requires.

**Linux system administration.** I manage Linux-based server environments with a focus on automation. I write Bash and Python scripts for routine administration tasks, configure services, manage users and permissions, and monitor system health. I have a strong preference for understanding what is happening at the OS level rather than treating the server as a black box.

**Infrastructure tooling.** I build CLI tools and automation scripts that reduce operational friction — from development environment provisioning to deployment helpers. I prefer tools that do one thing well and compose with existing workflows rather than monolithic platforms that try to do everything.

## How I Work

I keep infrastructure declarative wherever possible. Docker Compose files, CI/CD configurations, and deployment scripts are checked into version control alongside the application code. If a setup cannot be reproduced from the repository alone, it is not done.

I value simplicity in infrastructure. A well-written shell script is often better than a complex orchestration platform for the problems I solve. I reach for heavier tools only when the problem genuinely requires them — not because they are popular.

I automate the things that are boring, error-prone, or both. Manual deployment processes, environment setup, and repetitive server tasks are all candidates for automation. The goal is a workflow where developers focus on code and the pipeline handles everything else.
