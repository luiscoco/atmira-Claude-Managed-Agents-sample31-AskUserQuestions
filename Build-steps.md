# AskUserQuestion

This file describes the **steps followed** to add Concept 31 (**AskUserQuestion**) to the Claude Agent SDK Lab: what
was read, what was probed, what was decided, how it was tested, and what the tests changed. To learn the concept
itself, read [Tab31-AskUserQuestion.md](Tab31-AskUserQuestion.md) (the same text is in `readme.md`).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 31 | The agent asks the user multiple-choice questions: a host dialog in `canUseTool`, every kind of answer (pick, multi-select, free text, note, skip, cancel, timeout), HTML previews, a hook that answers, a partial answer, when the tool is removed | `/api/c31/tools`, `/run`, `/answer`, `/code` | [Tab31-AskUserQuestion.md](Tab31-AskUserQuestion.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample30, so `npm install` is not needed. Open the **31. AskUserQuestion** tab.
`ANTHROPIC_API_KEY` must be in `.env`, and the account must have credit (see Step 9). Start it from a normal terminal,
not from inside Claude Code (see Tab16). Each scenario costs about $0.03 or less.

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text, as in sample28 to sample30. A grep of
the `Tab*.md` files and of `server/` and `src/` found no mention of `AskUserQuestion`, `ExitPlanMode`, OpenTelemetry,
MCP elicitation or `spawnClaudeCodeProcess`. Four topics were offered, and **AskUserQuestion** was chosen.

## Step 2: Read what the course already said

sample31 held a copy of sample30 without `node_modules`, so `node_modules` was copied from sample30.

| Read | To learn |
|---|---|
| `Build-steps.md`, `30-todo-tracking.ts`, `Concept30TodoTracking.tsx` | The latest style: strict zod bodies, `relay()`, `#region` + `/code`, scenario buttons with hints, a setups table (button 0), a fake `CLAUDE_CONFIG_DIR`, a folder per run, a host check, `short()` for paths |
| `server/index.ts`, `server/sse.ts`, `src/lib/sse.ts`, `src/App.tsx`, `src/styles.css` | Mounting, SSE, the tab list, the CSS classes to reuse (`card permission`, `tool-call`, `tag-*`) |
| `Tab30-Todo-tracking.md`, `Tab1-query().md` | The explanation's structure, and the table row format |

## Step 3: Check the types

In `sdk-tools.d.ts` and `sdk.d.ts` (`0.3.281`):

| Found | Used for |
|---|---|
| `AskUserQuestionInput`: 1–4 `questions` with `question`, `header` (≤ 12 chars), 2–4 `options` (`label`, `description`, `preview?`), `multiSelect`; also `answers?`, `annotations?`, `metadata?` | The card, and what `updatedInput` must hold |
| `AskUserQuestionOutput`: `answers` ("multi-select answers are comma-separated"), `response?`, `annotations?` (`notes`, `preview`), `afkTimeoutMs?` | The multi-select format, notes, `tool_use_result` |
| `Options.toolConfig.askUserQuestion.previewFormat: 'markdown' \| 'html'` | Scenario 3 |
| `CanUseTool` (`signal`, `toolUseID`, `title`, `displayName`), `PermissionResult` (`deny` + `interrupt`) | `ask()`, skip and cancel |
| `permissionPrompts: 'host' \| 'none'`, the `askUserQuestionTimeout` setting | Button 0; the host's own timeout |

## Step 4: Probe before designing

Scratchpad scripts (in a temporary `.probe/` folder) ran real sessions with Haiku 4.5, each with its own
`CLAUDE_CONFIG_DIR`, and printed each tool call, `canUseTool` call and tool_result:

| Probe | Result | Decision |
|---|---|---|
| Answer from `canUseTool` with `updatedInput.answers` | "Your questions have been answered: …"; `tool_use_result` = `{ questions, answers }` | The core of the lab |
| No `canUseTool` / `permissionPrompts: 'none'` / `bypassPermissions` without `canUseTool` | The tool is **not in `system/init`**; the model asks in plain text | Button 0 |
| `deny` with a message | `is_error: true` with the message; the model used defaults; `permission_denials` lists it | Skip, and the host timeout |
| `deny` + `interrupt: true` | A fixed "STOP…" text, `error_during_execution`, then `query()` throws | Cancel; `runOnce()` catches |
| Free text; a note in `annotations` | The wording switches to "The user answered… Read the answers carefully"; the model asked back | Part D table |
| `PreToolUse` hook with `updatedInput.answers` | It answers; `canUseTool` is not called; lowercase `"green"` also gets the "carefully" wording | Scenario 5 |
| Answer only one of two questions | The model **made up** the second answer | Scenario 6; `/answer` refuses partial answers |
| "Write a poem" (Haiku, Sonnet 5, with and without a system prompt append) | Never asked | Scenario 4; every prompt asks for questions |
| `previewFormat: 'html'` / default | HTML fragments / Markdown in `preview` | Scenario 3, sandboxed iframe |
| `canUseTool` waits 150 s | The run goes on normally: no limit | A 2-minute host timeout |
| Default tool set | Called directly, no `ToolSearch`: not deferred | Step 4 of the explanation |
| A subagent asked to ask | Unclear: the agent was launched in the background, and the main agent asked itself | Left out |

## Step 5: Design the concept

- **A pending question per call.** `ask()` puts a `{ questions, input, done }` entry in a map keyed by
  `<run id>:<question id>`, emits a `question` SSE event, and returns a promise. `POST /answer` validates the body,
  builds the `PermissionResult` (`toResult()`) and calls `done`. A timer (2 min), the run's abort `signal` and the end
  of the run also call `done`, so no promise is left open.
- **One router, four routes.** `POST /tools` (6 setups in parallel), `POST /run` (7 scenarios: `ask`, `multi`,
  `preview`, `unprompted`, `hook`, `partial`, `custom`), `POST /answer`, `GET /code`.
- **A folder per run** (`ask-lab/runs/<id>/`, empty), and one fake `CLAUDE_CONFIG_DIR` (`ask-lab/config`).
- **The card**: radio or checkboxes, an "Other" text field added by the host, a note field, the option's preview
  (hover), a countdown, Submit (disabled until complete), Skip, Cancel the job.
- **The host check**: every question with the answer sent, or a red "no answer" row.
- **Part C**: `/code` regions `options`, `ask`, `answer`, `hook`, `messages`, `check`. **Part D**: the answers table.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/31-ask-user-question.ts` | New: `newRun()`, `baseOptions()`, `ask()` / `record()`, `AnswerBody` + `toResult()` + `/answer`, `answerFromProfile()` (the hook), `relay()`, `check()`, the routes |
| `server/index.ts` | Mounted on `/api/c31` |
| `src/concepts/Concept31AskUserQuestion.tsx` | New: `QuestionCard`, `Preview`, `Timeline`, `ToolsTable`, Parts A to D, the custom job form |
| `src/App.tsx`, `src/styles.css` | Tab; `.tag-ask`, `.tool-call.ask`, `.ask-*` |
| `.gitignore`, `Tab1-query().md` | `ask-lab/`, table row |

`npx tsc --noEmit -p .` passed.

## Step 7: Test the routes

Port 3001 was left free for the other samples. A scratchpad script (`serve.ts`) mounted **only** the Concept 31 router
on port **3131**, with `.env` loaded and the `CLAUDE*` variables removed. A second script (`drive.ts`) posted each body
the way the tab does, printed each SSE event on one line, and answered each `question` event with `POST /answer`.

| Test | Result |
|---|---|
| `{ scenario: "rm" }`, `{ scenario: "ask", prompt: "x" }`, `{ scenario: "custom" }`, `{ scenario: "ask", previewFormat: "html" }` | Refused with a clear message, nothing started |
| `/answer`: one of two answers; an unknown question; `action: "rm"`; a wrong id; skip twice | 400, 400, 400, 404, then 200, then 404 |
| `POST /tools` | 6 rows (see Step 8) |
| `ask`, `multi`, `preview` answered | 2/2, 2/2 (`"Pepperoni, Mushrooms"`), 1/1; files written |
| `ask` with free text + a note; skip; cancel | "The user answered… notes: make it funny"; defaults used; `error_during_execution` + the thrown error, caught |
| `hook`, `partial`, `unprompted` | Answered by the hook; the model wrote a made-up "Favorite Fruit: Pear", flagged by the host check; no question |

## Step 8: What the tests changed

- **`bypassPermissions` does NOT remove the tool.** In the first probe it had no `canUseTool`, so it looked like it
  did. Button 0 showed the tool listed with `canUseTool` + `bypassPermissions`. A new probe then printed the SDK's
  warning: *canUseTool will not be invoked: permissionMode 'bypassPermissions' auto-approves every tool call*. The code
  comment, the Part D table and the explanation were corrected.
- **A hook alone does not bring the tool back.** A `PreToolUse` hook without `canUseTool` → not listed. Added to the
  table and to Step 7 of the explanation.
- **An API error looked like "the model asked nothing".** With the key out of credit, the tab showed `result success`
  and the host check "The model asked nothing". As in Concept 28, `relay()` now reads `error` on the synthetic
  assistant message Claude Code writes when the API fails (`billing_error` here). The tab shows an **API error** row,
  marks the result, and the host check says the model never ran. Checked with the out-of-credit key (free).

## Step 9: Run it in the real app

The API account ran out of credit ("Credit balance is too low") during the `bypassPermissions` probe. So the model's
behavior with `bypassPermissions` after the warning was **not tested**, and the browser test could not use real runs.
Instead, a scratchpad mock (`mock.ts`) on port 3131 served a hand-written `/run` stream (one question with 3 HTML
previews, one of them with a `<script>`, and one `multiSelect` question) and a mock `/answer`, while the real router
served `/code`. Vite ran on port **5199** with a scratchpad config whose proxy pointed to 3131, and headless Edge was
driven through the DevTools protocol (`shot.mjs`, its profile in the scratchpad):

| Check | Page |
|---|---|
| Open tab 31, click scenario 3 | The card appears; Submit is disabled; the 6 code buttons are there |
| The preview iframe | `sandbox=""`; the page cannot read into it |
| Pick "Bold", check "Hero" and "Features", type a note | The preview follows the pick; Submit is enabled |
| Submit | Body `answers: { style: "Bold", sections: "Hero, Features" }, notes: { style: "dark mode please" }`; the card closes; host check 2/2 |
| Console | No error |

All test processes were stopped, and `ask-lab/` and `.probe/` were deleted.

Costs: about $0.60 for all the probes and route tests.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/31-ask-user-question.ts` | New: the Concept 31 routes, the pending questions, the hook |
| `server/index.ts` | Mounts `/api/c31` |
| `src/concepts/Concept31AskUserQuestion.tsx` | New: the AskUserQuestion tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Question card and row styles |
| `.gitignore` | `ask-lab/` |
| `Tab1-query().md` | Adds Concept 31 to the table |
| `Tab31-AskUserQuestion.md` | Explanation of the concept |
| `readme.md` | Same content as `Tab31-AskUserQuestion.md` |
| `Build-steps.md` | This file |
