+++
draft = false
date = '2026-03-14T16:18:18+07:00'
title = 'Appium Veronika Server'
type = 'project'
description = 'A REST API server for automating Veronika chatbot testing on the MyTelkomsel Android app using Appium and WebDriverIO. Manages WebDriver sessions, executes test cases on real devices, and captures responses with screenshots.'
image = ''
repository = 'https://github.com/mnabila/appium_veronika_server'
languages = ['typescript']
tools = ['appium', 'hono', 'webdriverio', 'docker', 'nodejs']
+++

This is the mobile counterpart to the [Playwright Veronika Server](/project/playwright-veronika-server/). While that project automates Veronika through a web browser, this one targets the actual MyTelkomsel Android app — interacting with the Veronika chat assistant on a real device (or emulator) through Appium and WebDriverIO.

## Why Appium?

Testing a chatbot through the web interface is useful, but it doesn't cover the mobile experience. The MyTelkomsel app has its own implementation of Veronika with native Android UI elements. To properly test chatbot responses in the mobile context, you need to drive the actual app on an actual device. That's where Appium comes in — it provides the WebDriver protocol for Android automation, and WebDriverIO makes it ergonomic to use from TypeScript.

## How It Works

The server connects to an Appium server that has the UiAutomator2 driver installed. When a session is created, it launches the MyTelkomsel app on the specified Android device, navigates through the app UI to reach the Veronika chat screen, and keeps the session alive for subsequent test cases.

The test execution flow:

1. **Create a session** — connects to Appium, launches MyTelkomsel, and navigates to the Veronika chat
2. **Run a test case** — taps the keyboard, types a question, sends it, and waits for a response matching a keyword
3. **Capture results** — returns the response text, a screenshot of the chat screen, and timing data
4. **Clear chat** — optionally clears the conversation history between test runs
5. **Clean up** — terminates the app, force-stops it via ADB, and closes the WebDriver session

## API Endpoints

The full API specification is available in the [`openapi.yaml`](https://github.com/mnabila/appium_veronika_server/blob/main/openapi.yaml) file. All endpoints are served under `/api/v1`:

| Method   | Endpoint                      | Description                                          |
|----------|-------------------------------|------------------------------------------------------|
| `POST`   | `/session/create`             | Initialize a new WebDriver session on an Android device |
| `GET`    | `/session/{sessionId}`        | Retrieve session details                             |
| `DELETE` | `/session/{sessionId}`        | Terminate app, close session, and remove from memory |
| `DELETE` | `/session/{sessionId}/clear`  | Clear the Veronika chat history via UI interaction   |
| `POST`   | `/process`                    | Send a message and capture Veronika's response       |

## Request Validation

All incoming requests are validated using [Zod](https://zod.dev/) schemas. Invalid payloads get a structured `400` response with detailed validation errors, making it easy to debug issues from the client side.

## Project Structure

```
src/
  index.ts              — Entry point, server setup with Hono
  router/
    index.ts            — Route definitions
  controller/
    session.ts          — Session CRUD handlers
    veronika.ts         — Test case execution handler
  usecase/
    session.ts          — WebDriver session lifecycle (Appium)
    veronika.ts         — Chat interaction logic (UiAutomator2 selectors)
  dto/
    index.ts            — Zod schemas and type definitions
  util/
    index.ts            — Date formatting utilities
```

## Prerequisites

- Node.js 20+
- An Appium server running with the UiAutomator2 driver
- An Android device or emulator connected via ADB
- MyTelkomsel app installed on the device

## Running It

For local development:

```bash
npm install
npm run dev
```

For production:

```bash
npm run build
npm start
```

The server starts at `http://localhost:3000`.

## Docker

The project includes a multi-stage Dockerfile — first stage compiles TypeScript, second stage runs only the production build with minimal dependencies:

```bash
docker build -t appium-veronika-server .
docker run -p 3000:3000 appium-veronika-server
```

Note that the Docker container still needs network access to an Appium server and an Android device connected via ADB.

## Tech Stack

- **Hono** — lightweight web framework for routing and middleware
- **WebDriverIO** — WebDriver client for communicating with Appium
- **Appium + UiAutomator2** — Android UI automation
- **Zod** — runtime request validation with TypeScript type inference
- **TypeScript** — type safety across the entire codebase
- **Docker** — containerized deployment
