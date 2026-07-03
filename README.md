# Agnes Video WebUI Demo

Local single-file WebUI demo for Agnes Video V2.0 batch generation workflows.

## What It Does

- Manage multiple Agnes API keys locally.
- Run one task per enabled key for parallel generation.
- Automatically serialize multiple jobs when only one key is enabled.
- Split one prompt into multiple jobs.
- Create and poll Agnes Video V2.0 tasks.
- Show completed video results and open returned video links.

## Privacy

- API keys are entered in the browser and kept in the current page session.
- Plaintext keys are not written to `localStorage`.
- This demo does not include any real API key.
- Validation outputs, local probe data, and generated videos are ignored by Git.

## Usage

Open `outputs/agnes-video-webui-demo.html` directly in a browser.

The demo supports:

- Mock mode for UI and scheduling checks.
- Real API mode for local Agnes API testing.

## Current Scope

This version intentionally removes browser download behavior. Completed jobs are listed with an `打开视频` action only.
