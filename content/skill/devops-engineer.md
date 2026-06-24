+++
draft = false
date = '2025-08-24T11:54:19+07:00'
title = 'DevOps Engineer'
type = 'skill'
description = 'CI/CD pipelines, Docker, and Linux server management and make sure code gets from a developer machine to production without drama.'
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

I set up CI/CD pipelines, containerize services, and manage Linux servers. The goal is always the same: get code to production without someone having to SSH in and do things by hand.

## What I Do

**CI/CD pipelines.** I use GitLab CI/CD and GitHub Actions. A typical pipeline builds, tests, lints, containerizes, and deploys, in that order, stopping at the first failure. I spend time on caching so builds stay fast, and on error messages so failures are obvious.

**Docker and containers.** Multi-stage builds to keep images small, Docker Compose for local dev environments so the whole team runs the same stack. I use Podman too. When the official image doesn't have what I need (like the RabbitMQ delayed message exchange plugin), I build a custom one.

**Linux servers.** I manage Linux servers directly, configuring services, writing Bash and Python scripts for routine tasks, setting up users and permissions. I run Arch Linux daily and have administered Ubuntu and Debian servers. I prefer knowing what the OS is actually doing over hiding behind abstractions.

**CLI tools and scripts.** When something is tedious or error-prone, I write a tool for it. Dev environment provisioning, deployment helpers, log collection. Small scripts that fit into existing workflows.

## How I Work

Everything goes in version control. Docker Compose files, CI/CD configs, deployment scripts, all of it lives in the repo next to the application code. If I can't reproduce a setup from the repository alone, it's not finished.

I keep things simple. A shell script that works is better than a platform I have to fight with. I'll reach for heavier tools when the problem actually calls for it, but most of the time it doesn't.

If a developer has to do something manually more than a few times, it should be automated. Deployments, environment setup, server maintenance that's what pipelines and scripts are for.
