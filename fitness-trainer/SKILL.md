---
name: fitness-trainer
description: Complete reference for Rune's AI Fitness Trainer solution. Use this skill whenever the user wants to update, fix, add features to, or debug the fitness trainer — including changes to workout programs, exercises, Telegram bot behaviour, n8n workflows, Google Sheets structure, Home Assistant integration, or GIF/image library. Trigger even for small requests like "add an exercise", "change the reminder time", "fix the bot", or "update the program". This skill gives you all IDs, credentials, workflow structure, and rules so you don't have to re-fetch them.
---

# Fitness Trainer — Complete Reference

## n8n access
- Base URL: `https://n8n.keldsen.org/api/v1`
- API key: in `~/.claude/mcp.json` under `N8N_API_KEY`
- Always use HTTP requests directly (n8n MCP tools return empty)
- Activate workflows via `POST /workflows/{id}/activate` (not PATCH)
- PUT body: only `name`, `nodes`, `connections`, `settings` — strip all other top-level fields

---

## Workflows

| ID | Name | Active | Cron |
|----|------|--------|------|
| `h4ecKU62SDdY8J4k` | Fitness Trainer - Telegram Handler | ✅ | webhook |
| `Z6jzuFpCxaEpvj5W` | Fitness Trainer - Daily Workout Scheduler | ✅ | `0 7 * * *` |
| `7Pk438PiPnKOW6rz` | Fitness Trainer - Training Reminder | ✅ | `0 18 * * 0-4` |
| `JwRcEJ4hk18cOBs3` | Fitness - Reset workout duration at midnight | ✅ | midnight |
| `jgbMu0U9z0E7dqQq` | Fitness Trainer - HA Data Webhook | ✅ | GET webhook |

---

## Telegram Handler (`h4ecKU62SDdY8J4k`) — the main workflow

### Trigger
- Node: `Telegram Trigger` (webhookId `7aa469fb-1d2e-4b1b-9ec0-d31f90f21ea6`)
- Listens to: `message`, `callback_query`
- Credential: `hZxxpbQ2w2WeNqp1` (name: "Fitness Trainer")
- Rune's chat ID: `8249159439`

### Deduplication
- Node `Deduplicate Update` (Code, runs right after trigger)
- Uses `$getWorkflowStaticData('global').seenUpdateIds` (array, max 200)
- Returns empty array if duplicate → stops execution

### Routing
**By message type** (`Route by Type` Switch):
- out0 Photo → equipment scan (GPT-4o vision → Google Sheets `equipment`)
- out1 Voice → transcribe → AI intent
- out2 Text → AI intent
- out3 Callback → callback handler

**By AI intent** (`Route by Intent` Switch, after GPT-4o `Detect Intent`):
| out | Intent | Description |
|-----|--------|-------------|
| 0 | LOG_SET | user reported exercise result |
| 1 | SESSION_DONE | user finished workout |
| 2 | GET_WORKOUT | show today's plan |
| 3 | GET_PROGRESS | progress report |
| 4 | START_TRAINING | begin session |
| 5 | HOW_TO | show GIF for exercise |
| 6 | CHAT | general conversation |

**By callback** (`Route Callback` Switch):
| out | callback_data | Action |
|-----|--------------|--------|
| 0 | `yes_start_training` | show today's plan |
| 1 | `no_not_today` | rest day reply |
| 2 | starts with `done\|` | log exercise from button |
| 3 | starts with `change\|` | weight-only change prompt |

### Callback data formats
```
done|exerciseName|sets|reps|weight|muscleGroup   → log exercise
change|exerciseName|sets|reps|muscleGroup        → prompt for new kg only
yes_start_training                               → from daily reminder
no_not_today                                     → skip today
```

### Warm-up flow (START_TRAINING)
- `Find Start Exercise`: if `today_logged` is empty → outputs `__warmup__` exercise
- Warmup message: "🏃 *Warm-up: 10 min crosstrainer*"
- Button sends callback `done|__warmup__|1|1|0|muscleGroup`
- After Merge for Done → `Is Warmup?` If node checks if callback contains `__warmup__`
  - true → `Find First from Warmup` (finds first real exercise, no logging)
  - false → `Log Exercise from Button` (normal log path)

### Weight-only change flow (change button)
- Button callback: `change|exerciseName|sets|reps|muscleGroup`
- `Parse Change Context` stores exercise in `staticData.pendingChanges[chatId]`
- Prompt sent: "✏️ How many kg for *ExerciseName*?" (rendered via `={{ "✏️ How many kg for *" + $json.exercise_name + "*?" }}`)
- Next text message: `Build AI Context` checks static data → sets `should_fast_log: true` if pending change exists AND message is a number
- `Has Pending Change?` If node: true → `Log Weight Change` (reads static data, logs with original sets/reps + new weight, clears pending) → `Save Log Entry` → `Find Next Exercise`

### Static workflow data (persists across executions)
```js
$getWorkflowStaticData('global')
  .seenUpdateIds    // string[] — dedup update_ids (max 200)
  .pendingChanges   // {[chatId]: {exerciseName, sets, reps, muscleGroup}}
```

### GIF lookup pattern
For both start and next exercise, the flow is:
`Find Exercise → Has Exercise? → Read GIFs → Lookup GIF → Has GIF?`
- GIF found: `Send GIF (sendAnimation)` → then send text+buttons
- No GIF: send text+buttons directly

### Session done flow
`Prepare Session Summary → Update Muscle Tracking (Sheets) + Update PR Entities → Call HA PR Update → Reply Session Done`

---

## Daily Workout Scheduler (`Z6jzuFpCxaEpvj5W`)

Runs at 07:00 daily. Determines least-recently-trained muscle group, generates workout via GPT-4o, saves to `programs` sheet, sends Telegram message.

### System prompt rules (in `Generate Workout Plan` node)
- ALWAYS include crosstrainer warm-up in `plan_text` — NOT in `exercises` JSON array
- Cable machine for majority of exercises (3–4 out of 5)
- Always kg, never lbs
- **NEVER use Dumbbell Hammer Curl** → always use **Cable Rope Hammer Curl** (low pulley, rope attachment, neutral grip) — same weight as starting point
- Weight rules: use exact weights from last session; progressive overload only if all sets completed
- Muscle groups: Back+Biceps, Chest+Triceps, Legs+Core, Shoulders+Arms

### Equipment (as known to the AI)
- Gymleco 225 Cable Cross (full cable crossover — high/low pulleys both sides)
- Dumbbells and kettlebells
- Adjustable bench
- Crosstrainer (warm-up only)
- Pull-up bar (integrated into cable machine top)
- Exercise mat

---

## Training Reminder (`7Pk438PiPnKOW6rz`)

- Runs Mon–Fri at 18:00
- Checks `person.rune_keldsen` HA entity → only sends if `state == 'home'`
- Checks if already trained today (reads `workout_log`)
- Sends Telegram with Yes/No inline buttons (`yes_start_training` / `no_not_today`)

---

## HA Data Webhook (`jgbMu0U9z0E7dqQq`)

- GET `https://n8n.keldsen.org/webhook/fitness-dashboard`
- Returns JSON: muscle tracking, today's plan + exercises, recent logs, GIF URLs
- Used by the HA dashboard card

---

## Google Sheets

**Spreadsheet ID:** `1PkASuD9ipjB72h5cxX7Rfo6SltnhwwiRhyjcOhCM3HY`
**Credential:** `022silQR0RZWrfMe` (name: "Google Sheets account")

| Sheet ID | Name | Columns |
|----------|------|---------|
| `0` | `equipment` | name, type, max_weight, notes, chat_id |
| `1` | `workout_log` | date, muscle_group, exercise, set_num, reps, weight, notes, chat_id, response_message, exercises_json, today_logged, just_logged |
| `2` | `muscle_tracking` | muscle_group, last_trained, session_count |
| `3` | `programs` | date, muscle_group, program_json, exercises_json, telegram_message |
| `exercise_gifs` | `exercise_gifs` | exercise_name, gif_url, img_url |

### exercise_gifs sheet
- `gif_url` is used for animations (sendAnimation node)
- `img_url` is a still image fallback (not always populated)
- When adding a new exercise to the program, add it to `exercise_gifs` too (gif_url can be empty)
- GIF lookup is case-insensitive on `exercise_name`

### n8n Sheets gotchas
- To read rows: omit `operation` field entirely (`"getRows"` and `"getAll"` both error)
- `appendOrUpdate` for upsert, `append` for new rows
- After a Sheets append node, only sheet columns survive in output — read upstream context via `$('NodeName').first()`

---

## Home Assistant integration

### PR entities (updated after each session via `input_number.set_value`)
```
input_number.fitness_bench_press_pr
input_number.fitness_chest_fly_pr
input_number.fitness_shoulder_press_pr
input_number.fitness_bicep_curl_pr
input_number.fitness_tricep_pushdown_pr
input_number.fitness_lateral_raise_pr
input_number.fitness_upright_row_pr
input_number.fitness_squat_pr
input_number.fitness_deadlift_pr
```
Updated via `Call HA PR Update` (homeAssistant node, credential `7f6R7EYBfP7HzE4D`, service `input_number.set_value`).

### Other HA entities used
- `person.rune_keldsen` — presence check for training reminder

---

## Key patterns and gotchas

- **Prefer homeAssistant node over httpRequest** for all HA calls (credential `7f6R7EYBfP7HzE4D`)
- **After Merge (combineAll)**, use `$('NodeName').all()` in Code nodes — `$input.all()` collapses and loses source rows
- **n8n webhook nodes need a static `webhookId`** UUID — without it, the webhook path breaks
- **To run a one-shot operation**: create workflow with webhook trigger + static webhookId, activate via `POST /workflows/{id}/activate`, call the webhook, then deactivate + delete
- **Duplicate Telegram messages** were caused by Telegram webhook retries (slow AI chain) — fixed by the `Deduplicate Update` node using static data
- **n8n OpenAI v2**: use `operation: "response"` (not "message") for `resource: "text"`; output at `$json.output[0].content[0].text`
- **Shared downstream nodes must handle all upstream paths**: Any Code node reached from multiple paths (e.g. warmup path vs normal path) must try each possible upstream node with `try/catch` fallbacks. Example: `Lookup GIF for Next` must try both `$('Find Next Exercise')` and `$('Find First from Warmup')` — whichever path ran.
- **`Find Next Exercise` context sources**: This node is reached from 3 paths — try in order: `$('Expand Log Sets')`, then `$('Log Exercise from Button')`, then `$('Log Weight Change')`. All three output `{ chat_id, exercises_json, today_logged, just_logged }`.
- **Telegram node text expressions**: `{{ $json.field }}` in a text field WITHOUT a leading `=` renders as literal text. Always use `={{ "prefix" + $json.field + "suffix" }}` for mixed strings, or `={{ $json.field }}` for pure expressions.

---

## Workflow update checklist

When modifying a workflow:
1. `GET /api/v1/workflows/{id}` — fetch full JSON first
2. Make targeted changes to `nodes` and/or `connections`
3. `PUT /api/v1/workflows/{id}` with body `{name, nodes, connections, settings}` only
4. Verify the response includes your changed nodes
5. If adding a new exercise, also add it to `exercise_gifs` sheet
6. If changing program rules, update the system prompt in `Generate Workout Plan` (Daily Scheduler)
