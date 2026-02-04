---
name: validate
description: Run the /validate quality-control slash command: spawn a low-cost subagent to run swift build, skip android build, and SwiftLint (make lint) with reporting. Use when a user asks to validate, check build+lint, run QC, or validate a specific package in this Skip monorepo.
---

# Validate

## Overview

Run a lightweight validation workflow by delegating the actual build and lint commands to a fast subagent, then report back success or failure with concise details.

## Workflow

### 1) Main agent: spawn a worker subagent

- Always run `/validate` by spawning a `worker` subagent so the main agent stays responsive.
- Pass the target argument (if any) exactly as provided by the user.
- Ask the subagent to stream progress and return a final summary (success/failure + key details).

### 2) Subagent: resolve target package (optional)

Interpret `/validate` with an optional target:
- No argument: use repo root (`/Users/hugo/Swishly/Projects/webvideocast/platforms/skip`).
- Argument provided:
  - If it is a valid path (absolute or relative to repo), use it.
  - Else try directory names in order: `libs/<name>`, `libs_os/<name>`, `webtv-sandbox` (if name matches).
  - Else search for directories containing `Package.swift` and match by folder name.
  - If multiple matches or none found, report back to main agent and stop.

Require a `Package.swift` in the resolved directory before running builds.

### 3) Subagent: run builds

Run the following from the resolved directory, in order:
1. `swift build`
2. `skip android build`

If `skip android build` fails and the error suggests device issues, run `adb devices` and include the device list in the report.

### 4) Subagent: run lint on recently updated code

- Gather changed files:
  - `git status --porcelain`
  - `git diff --name-only --cached`
- Filter to Swift files. If a target package was specified, filter to files under that directory.
- If any Swift files changed, run `make lint` from `webtv-sandbox/` (repo-relative).
- If no Swift files changed, skip lint and say so explicitly.

### 5) Subagent: report results to main agent

- On success: report each command as OK, note whether lint ran or was skipped, and list changed Swift files (truncate if long).
- On failure: report the failing command, exit code, and the last ~80 lines of output. Note which remaining steps were skipped.

## Notes

- Keep output concise and focused on actionable details.
- If the target directory is missing or ambiguous, request clarification before running any commands.

## Examples

- `/validate`
- `/validate skip-tca`
- `/validate libs/skip-tca`
