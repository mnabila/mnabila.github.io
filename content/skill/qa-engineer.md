+++
draft = false
date = '2025-08-24T11:54:13+07:00'
title = 'QA Engineer'
type = 'skill'
description = 'Building test automation infrastructure with Playwright and Appium — turning manual QA workflows into API-driven pipelines.'
languages = ['javascript', 'golang']
tools = [
  'appium',
  'uiautomator',
  'webdriverio',
  'playwright',
  'k6'
]
+++

My QA work is less about writing test cases and more about building the systems that run them. If the team is spending hours doing something manually in a browser or on a phone, I'd rather spend a few days building a server that does it through an API call.

## What I Do

**Test automation servers.** I build REST APIs that wrap browser and mobile automation. For example, I built two servers for testing Telkomsel's Veronika chatbot one using Playwright for the web version, another using Appium and WebDriverIO for the Android app. Both expose the same API: create a session, send a message, get the response and a screenshot. Any CI/CD pipeline or test script can call them without knowing what's happening underneath.

**Cross-platform coverage.** The Playwright and Appium servers share the same endpoint structure, so the test orchestration layer can switch between web and mobile by changing one URL. That was a deliberate choice so the QA team didn't need separate tooling for each platform.

**Load testing.** I use k6 to script user scenarios and measure how services hold up under traffic. Response times, throughput, error rates.

## How I Work

I treat test infrastructure like any other backend service. The automation servers I build have proper session management, resource cleanup (so browsers and device sessions don't leak), and element selectors that don't break every time someone moves a button.

The thing I've learned is that flaky tests are almost never a test problem they're an infrastructure problem. A browser instance that didn't get cleaned up, a session that timed out, a selector that was too brittle. Fixing those at the infrastructure level fixes them for every test at once.

When manual testing is eating up the team's time, I build the tooling to replace it. That usually pays for itself within a week.
