+++
draft = false
date = '2025-08-24T11:54:13+07:00'
title = 'QA Engineer'
type = 'skill'
description = 'Ensuring product quality through systematic testing, automation, and continuous validation across the software lifecycle.'
languages = ['javascript', 'golang']
tools = [
  'appium',
  'uiautomator',
  'webdriverio',
  'playwright',
  'k6'
]
+++

Quality assurance, for me, is not a phase at the end of development — it is infrastructure that runs continuously. My QA work focuses on building automation systems that make testing scalable, repeatable, and fast enough to run on every deployment.

## What I Do

**Test automation infrastructure.** I build the servers, APIs, and tooling that turn manual test workflows into automated pipelines. This includes browser-based automation with Playwright for web testing and native mobile automation with Appium and WebDriverIO for Android apps. I have designed session-managed automation servers that expose testing capabilities as REST APIs, allowing any CI/CD pipeline or test framework to consume them without custom integration work.

**Cross-platform testing.** I build test infrastructure that covers both web and native mobile platforms with a consistent API contract. The same test orchestration tools can target web or mobile by changing a base URL — the platform difference is an implementation detail, not a consumer concern.

**Performance testing.** I use k6 for load testing and performance validation, scripting realistic user scenarios and measuring response times, throughput, and error rates under sustained load.

## How I Work

I approach QA from an engineering perspective. The test infrastructure itself should be well-architected: clean separation between test logic and automation drivers, proper resource management for browser and device sessions, and API-first design so the automation layer integrates cleanly with whatever pipeline the team already uses.

I focus on making tests reliable, not just making them pass. That means investing in proper session lifecycle management, automatic cleanup of browser instances and device sessions, and resilient element selectors that survive minor UI changes without breaking the entire suite.

When manual testing is the bottleneck, I build the tooling to eliminate it. The highest-leverage QA investment is often not writing more test cases — it is building the infrastructure that lets existing test cases run without human intervention.
