# Autonomous Agent Specification: Zero-Token Macro Replay & Rate-Limit Pacing

## 1. Objective & Scope
Implement two self-contained capabilities in this Chrome Extension (Manifest V3):
1. **Deterministic Macro Recorder & Replay:** Record successful AI browser actions (clicks, inputs, navigations) into a reusable JSON workflow, and replay them directly via CDP/DOM selectors with **0 LLM API calls**, falling back to the LLM only if a selector fails.
2. **Gemini Free-Tier Rate-Limit Pacing:** Add configurable inter-step pacing (`stepDelayMs`) and automatic exponential backoff on `HTTP 429` errors so Google AI Studio free-tier keys do not fail mid-task.

### Strict Out-of-Scope Constraints
- **DO NOT** add Firefox support or remove `chrome.debugger` CDP calls.
- **DO NOT** add external backend servers, telemetry, or paid SaaS dependencies.
- **DO NOT** refactor unrelated UI components or change the existing package manager.

---

## 2. Environment & Verification Commands
- **Runtime:** Node.js >= 20, TypeScript (strict mode)
- **Install:** `pnpm install`
- **Build Extension:** `pnpm build` (must compile cleanly into `dist/`)
- **Typecheck:** `pnpm typecheck`
- **Run Unit Tests:** `pnpm test`

*Note: If running in a headless sandbox without a live Chrome GUI, verify all browser automation logic using unit tests and mock CDP debugger interfaces.*

---

## 3. Target Architecture & File Map
Create or update the following modules while keeping pure business logic decoupled from `chrome.*` APIs so it remains unit-testable:

- `src/types/workflow.ts` — Shared TypeScript interfaces for `RecordedStep`, `SavedWorkflow`, and `ReplayResult`.
- `src/core/rateLimiter.ts` — Exponential backoff wrapper (`withRateLimitRetry`) and configurable step delay helper.
- `src/core/workflowRecorder.ts` — Pure state collector that captures executed agent actions and serializes them to JSON.
- `src/core/workflowReplayer.ts` — Deterministic step runner that executes selectors directly and triggers single-step LLM self-healing on timeout/failure.
- `src/storage/workflowStorage.ts` — Wrapper around `chrome.storage.local` to save, list, export, and delete workflows.
- `src/sidepanel/components/WorkflowDrawer.tsx` — Minimal UI drawer in the side panel to save the last completed task and trigger one-click replay.
- `tests/workflowReplayer.test.ts` & `tests/rateLimiter.test.ts` — Vitest/Jest unit tests covering replay success, self-healing fallback, and 429 retry logic.

---

## 4. Data Contracts (Do Not Alter Signatures)

```typescript
export interface RecordedStep {
  index: number;
  action: 'click' | 'type' | 'navigate' | 'scroll';
  url: string;
  cssSelector?: string;
  ariaLabel?: string;
  fallbackText?: string;
  inputValue?: string;
  pressEnter?: boolean;
  description: string;
}

export interface SavedWorkflow {
  id: string;
  name: string;
  createdAt: number;
  steps: RecordedStep[];
}

export interface RateLimitConfig {
  stepDelayMs: number;      // Default: 2500 (safe for Google AI Studio free tier)
  maxRetries: number;       // Default: 4
  baseBackoffMs: number;    // Default: 3000
}
```

---

## 5. Execution Plan (Follow Sequentially)
1. **Inspect Existing Codebase:** Read `package.json`, `tsconfig.json`, and existing agent execution loop files to match existing naming conventions and test runners.
2. **Implement Rate-Limit Pacing (`src/core/rateLimiter.ts`):**
   - Wrap LLM provider calls so `HTTP 429` or `RESOURCE_EXHAUSTED` errors retry with exponential backoff (`baseBackoffMs * 2^attempt` + jitter).
   - Enforce `stepDelayMs` between consecutive agent turns when enabled in settings.
3. **Implement Recorder & Storage (`src/core/workflowRecorder.ts`, `src/storage/workflowStorage.ts`):**
   - Hook into the existing agent action executor to record every successful step's `cssSelector`, `ariaLabel`, `fallbackText`, and `inputValue`.
4. **Implement Deterministic Replayer (`src/core/workflowReplayer.ts`):**
   - Execute each `RecordedStep` directly against the active tab without calling the LLM.
   - Wait up to `3500ms` for the target selector to appear. If not found, invoke the existing single-step LLM agent to locate the updated element, update the saved `cssSelector` in storage, and continue replay.
5. **Wire Side-Panel UI (`WorkflowDrawer.tsx`):**
   - Add a **"Save Workflow"** button on completed agent runs.
   - Add a **"Saved Workflows"** list with **Run (0 Tokens)** and **Delete** buttons, plus a numeric input in Settings for `Step Delay (ms)`.
6. **Write Tests:** Add comprehensive unit tests for `rateLimiter.ts` (mocking 429 responses) and `workflowReplayer.ts` (mocking a broken selector triggering self-healing).

---

## 6. Definition of Done
Before concluding your task, you MUST verify all of the following:
- [ ] `pnpm typecheck` passes with zero errors.
- [ ] `pnpm test` passes all existing and newly added unit tests.
- [ ] `pnpm build` produces a valid Manifest V3 bundle in `dist/` with no missing imports.
- [ ] No API keys, `.env` files, or large binary assets are committed.
