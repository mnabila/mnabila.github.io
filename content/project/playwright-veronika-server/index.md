+++
draft = false
date = '2026-03-14T15:44:43+07:00'
title = 'Playwright Veronika Server'
type = 'project'
description = 'A headless browser automation server built with Hono and Playwright for interacting with Telkomsel Veronika chatbot. Features session management, REST API endpoints, and Docker support for scalable automated testing.'
image = ''
repository = 'https://github.com/mnabila/playwright_veronika_server'
languages = ['typescript']
tools = ['playwright', 'hono', 'docker', 'nodejs']
+++

This project was built to solve a specific problem: automating interactions with [Veronika](https://veronika.telkomsel.com/veronika), Telkomsel's virtual assistant chatbot, for testing purposes. Instead of manually opening a browser and typing messages one by one, this server handles everything programmatically using Playwright behind a clean REST API.

## The Idea

When you need to test chatbot responses at scale — verifying that specific questions return expected answers, checking keyword matches, or running regression tests — doing it manually just doesn't work. This server wraps Playwright's browser automation in a lightweight HTTP API, so any test framework or script can interact with Veronika by simply hitting an endpoint.

## How It Works

The server is built with [Hono](https://hono.dev/), a fast and lightweight web framework, running on Node.js. Under the hood, it launches headless Chromium instances via Playwright to interact with the Veronika chat interface just like a real user would.

The flow is straightforward:

1. **Create a session** — spins up a headless browser, navigates to Veronika, and keeps it ready
2. **Send a message** — posts a question to the chat and waits for Veronika's response
3. **Get the result** — returns the response text, a screenshot, and timing information
4. **Clean up** — delete the session when done

## API Endpoints

The full API specification is available in the [`openapi.yaml`](https://github.com/mnabila/playwright_veronika_server/blob/main/openapi.yaml) file. All endpoints are served under `/api/v1`:

| Method   | Endpoint               | Description                                          |
|----------|------------------------|------------------------------------------------------|
| `GET`    | `/health`              | Server status, session statistics, and uptime        |
| `POST`   | `/session/create`      | Create a new Chromium browser session                |
| `GET`    | `/session/list`        | List all active sessions                             |
| `GET`    | `/session/{sessionId}` | Get details for a specific session                   |
| `DELETE` | `/session/{sessionId}` | Close the browser and remove the session             |
| `POST`   | `/process`             | Send a message to Veronika and get the response      |

## Session Management

Each session represents an isolated browser instance. The `SessionManager` handles the lifecycle — creating sessions with unique UUIDs, tracking active requests, and automatically cleaning up idle sessions after a timeout period (default 2 minutes). This prevents browser instances from piling up and eating resources.

## Project Structure

```
src/
  index.ts              — Entry point, route definitions
  controller/
    session.ts          — Session CRUD handlers
    veronika.ts         — Chat processing handler
  service/
    browser/
      browser.ts        — BrowserSession class (Playwright lifecycle)
      session.ts        — SessionManager (session pool)
    veronika/
      index.ts          — VeronikaHelper (chat interaction logic)
  dto/
    veronika.ts         — Request/response type definitions
  util/
    datetime.ts         — Date formatting utilities
```

## Running It

For local development:

```bash
npm install
npx playwright install chromium
npm run dev
```

The server starts at `http://localhost:3000`.

## Docker

The project includes a multi-stage Dockerfile — first stage builds the TypeScript, second stage uses Microsoft's official Playwright image with Chromium pre-installed. This keeps the production image lean while having everything needed to run headless browsers:

```bash
docker compose up -d
```

## Tech Stack

- **Hono** — lightweight web framework with great TypeScript support
- **Playwright** — browser automation that handles all the chat interaction
- **TypeScript** — type safety across the entire codebase
- **Docker** — containerized deployment with Playwright's official image
- **PM2** — process management for production via `ecosystem.config.cjs`
