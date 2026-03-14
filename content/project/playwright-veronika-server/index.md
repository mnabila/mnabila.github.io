+++
draft = false
date = '2026-03-14T15:44:43+07:00'
title = 'Playwright Veronika Server'
type = 'project'
description = 'A deep dive into building a headless browser automation server that turns manual chatbot testing into scalable, API-driven test execution — built with Playwright, Hono, and TypeScript.'
image = ''
repository = 'https://github.com/mnabila/playwright_veronika_server'
languages = ['typescript']
tools = ['playwright', 'hono', 'docker', 'nodejs']
+++

Chatbot testing is deceptively labor-intensive. When the test scope is small, manual verification works fine. But the moment you need to validate hundreds of question-response pairs across regression cycles, manual testing becomes the bottleneck that holds back the entire QA pipeline.

This project is a headless browser automation server that wraps [Veronika](https://veronika.telkomsel.com/veronika) — Telkomsel's virtual assistant chatbot — behind a clean REST API. Instead of a human opening a browser and typing messages, any script or test framework can now interact with the chatbot through a single HTTP call.

## Problem Background

The QA team needed to validate Veronika's chatbot responses at scale: verifying that specific questions returned expected answers, checking keyword accuracy, and running regression suites after every deployment. The existing workflow was entirely manual — open a browser, navigate to Veronika, type a question, read the response, take a screenshot, log the result. Repeat.

For a handful of test cases, this was manageable. For hundreds, it was unsustainable. The team needed a programmatic interface that could integrate into existing test pipelines and CI/CD workflows without requiring human interaction.

## Solution Overview

I built a REST API server that manages headless browser sessions and exposes chatbot interaction as simple HTTP endpoints. Any test framework, script, or orchestration tool can create a browser session, send a message to Veronika, and receive the response — all through standard API calls.

**Tech stack:** TypeScript, Hono, Playwright, Docker, PM2

**My role:** Sole developer — architecture, implementation, API design, and containerization

## System Architecture

The server is built with [Hono](https://hono.dev/), a lightweight web framework running on Node.js. Under the hood, it manages headless Chromium instances via Playwright, simulating real user interactions with the Veronika chat interface.

The core workflow follows a session-based lifecycle:

1. **Create a session** — provisions a headless Chromium instance, navigates to Veronika, and holds it ready for requests
2. **Send a message** — submits a question to the chat interface and waits for the chatbot's response
3. **Retrieve the result** — returns the response text, a screenshot of the chat window, and timing metrics
4. **Clean up** — terminates the browser instance and releases all associated resources

The codebase follows a layered architecture with clear separation between routing, business logic, and browser management:

```
src/
  index.ts              — Entry point and route definitions
  controller/
    session.ts          — Session CRUD request handlers
    veronika.ts         — Chat processing request handler
  service/
    browser/
      browser.ts        — BrowserSession class (Playwright lifecycle)
      session.ts        — SessionManager (session pool and cleanup)
    veronika/
      index.ts          — VeronikaHelper (chat interaction logic)
  dto/
    veronika.ts         — Request/response type definitions
  util/
    datetime.ts         — Date formatting utilities
```

The full API specification is documented in [`openapi.yaml`](https://github.com/mnabila/playwright_veronika_server/blob/main/openapi.yaml). All endpoints are served under `/api/v1`:

| Method   | Endpoint               | Description                                          |
|----------|------------------------|------------------------------------------------------|
| `GET`    | `/health`              | Server health, session statistics, and uptime        |
| `POST`   | `/session/create`      | Provision a new Chromium browser session              |
| `GET`    | `/session/list`        | List all active sessions                             |
| `GET`    | `/session/{sessionId}` | Retrieve details for a specific session              |
| `DELETE` | `/session/{sessionId}` | Close the browser and release the session            |
| `POST`   | `/process`             | Send a message and capture the chatbot response      |

## Key Features

- **Session-based browser isolation** — each session runs its own Chromium instance, preventing state leakage between concurrent test runs
- **Automatic resource cleanup** — the `SessionManager` tracks idle sessions and terminates them after a configurable timeout (default: 2 minutes), preventing memory leaks under sustained load
- **Screenshot capture** — every response includes a screenshot of the chat window, providing visual evidence for test reports
- **Timing metrics** — each interaction records response latency, enabling performance monitoring alongside functional testing
- **OpenAPI specification** — the full API contract is documented in `openapi.yaml`, making client generation and integration straightforward
- **Multi-stage Docker build** — production images use Microsoft's official Playwright base with Chromium pre-installed, keeping the image lean while ensuring browser compatibility

## Technical Challenges and Solutions

**Browser resource management at scale.** The core challenge was managing multiple concurrent Chromium instances without leaking memory or orphaning processes. The solution was a `SessionManager` class that wraps each browser in a `BrowserSession` with UUID-based tracking, active request counting, and an idle timeout mechanism. When a session has no active requests for longer than the threshold, it is automatically terminated. This made the server stable under sustained test loads.

**Reliable chat response detection.** Veronika's chat interface does not provide a clean "response complete" signal. The Playwright interaction logic had to implement a polling strategy that watches for new message elements in the DOM and waits for the response to stabilize before capturing it. Getting the timing right — long enough to catch slow responses, short enough to keep tests fast — required careful tuning.

**Containerized Playwright.** Running Playwright inside Docker introduces its own set of challenges: Chromium requires specific system dependencies, and headless mode behaves differently across base images. Using Microsoft's official `mcr.microsoft.com/playwright` image as the production base resolved most compatibility issues, while the multi-stage build keeps the final image size manageable.

## Lessons Learned

**Build the tooling, not just the tests.** When a testing workflow is manual and repetitive, the highest-leverage investment is often building the infrastructure to automate it. The initial development time for this server paid for itself within the first week of use.

**API-first design enables integration.** By exposing chatbot interaction as a REST API rather than a standalone script, the tool became immediately usable by any test framework, CI/CD pipeline, or orchestration system the team already had in place. The integration cost for consumers was effectively zero.

**Session management is the hard part.** The chatbot interaction logic was straightforward. The real engineering challenge was lifecycle management — ensuring browsers get cleaned up, resources get released, and the server stays stable under concurrent load. In hindsight, this is where most of the design effort should have gone from the start.

## Conclusion

This project reduced chatbot regression testing from a multi-hour manual process to an automated pipeline that runs in minutes. The API-first approach made it trivial to plug into existing CI/CD workflows, and the session management layer ensures stable operation under sustained test loads.

For local development:

```bash
npm install
npx playwright install chromium
npm run dev
```

For containerized deployment:

```bash
docker compose up -d
```

The server starts at `http://localhost:3000`. The full source is available on [GitHub](https://github.com/mnabila/playwright_veronika_server).
