+++
draft = false
date = '2026-03-14T16:18:18+07:00'
title = 'Appium Veronika Server'
type = 'project'
description = 'A technical deep dive into extending chatbot test automation to native Android — driving the real MyTelkomsel app on physical devices with Appium, WebDriverIO, and a consistent API contract.'
image = ''
repository = 'https://github.com/mnabila/appium_veronika_server'
languages = ['typescript']
tools = ['appium', 'hono', 'webdriverio', 'docker', 'nodejs']
+++

Testing a chatbot through a web browser tells you how it behaves on the web. But when the majority of your users interact through a native mobile app, that coverage gap becomes a blind spot.

This project is the mobile counterpart to the [Playwright Veronika Server](/project/playwright-veronika-server/). Instead of automating a browser, it drives the actual MyTelkomsel Android app on physical devices and emulators — interacting with the Veronika chat assistant through native UI elements using Appium and WebDriverIO.

## Problem Background

The MyTelkomsel Android app has its own implementation of Veronika with native Android components. The chatbot's behavior, response rendering, and UI interactions can differ significantly from the web version. Issues that never surface in browser testing — such as keyboard handling, scroll behavior, or response rendering in a native view — can directly affect the mobile user experience.

The team had already solved web-based chatbot automation with the Playwright server. But mobile remained a manual process: a tester physically interacting with the app on a device, typing questions, reading responses, and logging results. To achieve true cross-platform test coverage, we needed the same API-driven approach on native Android.

## Solution Overview

I built a REST API server that manages Appium WebDriver sessions and exposes chatbot interaction on the MyTelkomsel app as standard HTTP endpoints. The same test orchestration tools that consume the Playwright server can target this server with minimal integration changes — the API contract is intentionally consistent across both platforms.

**Tech stack:** TypeScript, Hono, Appium, WebDriverIO, UiAutomator2, Zod, Docker

**My role:** Sole developer — architecture, Appium integration, API design, and containerization

## System Architecture

The server is built with [Hono](https://hono.dev/) and connects to an external Appium server running the UiAutomator2 driver. WebDriverIO serves as the client library for communicating with Appium, providing an ergonomic TypeScript API for driving Android UI elements.

The test execution lifecycle:

1. **Create a session** — establishes a WebDriver connection via Appium, launches the MyTelkomsel app on the target device, and navigates to the Veronika chat screen
2. **Execute a test case** — interacts with the on-screen keyboard, types a question, sends it, and waits for a response matching the expected keyword
3. **Capture results** — collects the response text, captures a device screenshot, and records timing data
4. **Clear chat** — optionally resets the conversation history via UI interaction for clean test state
5. **Clean up** — terminates the app, force-stops it via ADB, and closes the WebDriver session

The codebase follows a clean layered architecture:

```
src/
  index.ts              — Entry point, Hono server setup
  router/
    index.ts            — Route definitions
  controller/
    session.ts          — Session CRUD request handlers
    veronika.ts         — Test case execution handler
  usecase/
    session.ts          — WebDriver session lifecycle (Appium)
    veronika.ts         — Chat interaction logic (UiAutomator2 selectors)
  dto/
    index.ts            — Zod schemas and type definitions
  util/
    index.ts            — Date formatting utilities
```

The full API specification is documented in [`openapi.yaml`](https://github.com/mnabila/appium_veronika_server/blob/main/openapi.yaml). All endpoints are served under `/api/v1`:

| Method   | Endpoint                      | Description                                          |
|----------|-------------------------------|------------------------------------------------------|
| `POST`   | `/session/create`             | Initialize a new WebDriver session on an Android device |
| `GET`    | `/session/{sessionId}`        | Retrieve session details                             |
| `DELETE` | `/session/{sessionId}`        | Terminate app, close session, and release resources  |
| `DELETE` | `/session/{sessionId}/clear`  | Clear Veronika chat history via native UI interaction |
| `POST`   | `/process`                    | Send a message and capture the chatbot response      |

## Key Features

- **Real device testing** — drives the actual MyTelkomsel app on physical Android devices or emulators, capturing behavior that browser-based testing cannot detect
- **Consistent API contract** — intentionally mirrors the Playwright server's endpoint structure and response format, allowing test orchestration tools to target web or mobile without code changes
- **Zod request validation** — all incoming payloads are validated at the API boundary using [Zod](https://zod.dev/) schemas, providing both runtime safety and automatic TypeScript type inference. Invalid requests receive structured `400` responses with detailed error messages
- **Screenshot capture** — every test result includes a device screenshot for visual verification and test reporting
- **Chat state management** — dedicated endpoint for clearing conversation history between test runs, ensuring each test case starts with a clean state
- **Multi-stage Docker build** — TypeScript compilation and production runtime are separated into distinct build stages for minimal image size

## Technical Challenges and Solutions

**Native UI element identification.** Unlike web automation where you can query the DOM with CSS selectors, Android UI automation relies on UiAutomator2 selectors — resource IDs, class names, content descriptions, and XPath. The Veronika chat screen's element hierarchy was not designed with test automation in mind, which meant building resilient selectors that could survive minor UI updates without breaking the test suite.

**Keyboard and input handling.** Typing text into the chat input on Android involves interacting with the soft keyboard, which behaves differently across device models and Android versions. The interaction logic had to account for keyboard animation delays, input field focus state, and occasional input race conditions where characters would be dropped if sent too quickly.

**Session cleanup reliability.** A failed test should not leave orphaned app processes or stale WebDriver sessions on the device. The cleanup flow goes through multiple layers: terminating the app through Appium, force-stopping it via ADB as a safety net, and then closing the WebDriver session. This belt-and-suspenders approach ensures the device returns to a known state even when tests fail unexpectedly.

**Cross-platform API consistency.** Designing the API to be compatible with the Playwright server required careful abstraction. The chatbot interaction semantics are the same — send a message, get a response — but the underlying automation mechanisms are completely different (browser DOM vs. native Android views). The controller and usecase layers had to be structured so that the platform-specific logic stays encapsulated while the API surface remains identical.

## Lessons Learned

**Test where your users are.** Web-based testing is necessary but not sufficient. Native mobile apps have their own rendering, timing, and interaction characteristics. Real device testing caught issues — particularly around keyboard behavior and response rendering — that never appeared in browser automation.

**API contract consistency pays dividends.** By maintaining the same endpoint structure and response format across the Playwright and Appium servers, the team's test orchestration layer could target either platform by simply changing a base URL. The platform difference became an implementation detail, not a consumer concern. This was one of the most impactful architectural decisions in the project.

**Validate at the boundary, trust internally.** Using Zod at the API layer meant that every handler could assume its input was already validated and correctly typed. This eliminated defensive checks throughout the business logic and made the codebase significantly cleaner. The upfront investment in schema definitions saved time on every feature after.

**ADB is your safety net.** Appium's session cleanup is generally reliable, but edge cases exist — crashed apps, unresponsive devices, timeout scenarios. Adding ADB force-stop as a fallback in the cleanup flow eliminated a whole category of flaky test infrastructure issues.

## Conclusion

This project closed the mobile coverage gap in the team's chatbot testing pipeline. Combined with the Playwright-based web server, it provides a unified, API-driven approach to validating Veronika's responses across both platforms.

Prerequisites:
- Node.js 20+
- Appium server with UiAutomator2 driver
- Android device or emulator connected via ADB
- MyTelkomsel app installed on the target device

For local development:

```bash
npm install
npm run dev
```

For containerized deployment:

```bash
docker build -t appium-veronika-server .
docker run -p 3000:3000 appium-veronika-server
```

The server starts at `http://localhost:3000`. Note that the container requires network access to the Appium server and Android device. The full source is available on [GitHub](https://github.com/mnabila/appium_veronika_server).
