---
name: doko
description: Dokobot tools - read web pages, search the web, list connected dokos, and update skill definitions.
compatibility: Requires curl and DOKO_API_KEY environment variable.
allowed-tools: Bash
metadata:
  author: dokobot
  version: "1.0"
---

# Dokobot Tools

Dokobot API tools. All commands require `DOKO_API_KEY` environment variable.

**Usage**: `/doko <command> [arguments]`

Command: $ARGUMENTS[0]

## Prerequisites
- `DOKO_API_KEY` is set in environment (configure in `.claude/settings.local.json`)
- If no API Key is set, ask the user to create one at the Dokobot dashboard: https://dokobot.ai/dashboard/api-keys

## Commands

### read

Read a web page via the Chrome extension and return its content.

**Usage**: `/doko read <URL> <goal>`

**Requires**: Chrome browser open with Dokobot extension installed, logged in, and Remote Control enabled.

**Arguments**: URL = $ARGUMENTS[1], goal = remaining arguments after URL

```bash
curl -s --max-time 330 -X POST "https://dokobot.ai/api/tools/read" \
  -H "Authorization: Bearer $DOKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "<URL>", "goal": "<GOAL>", "maxDurationMs": 300000}'
```

**Response fields**:
- `data.text` — extracted page content as plain text
- `data.chunks` — structured content chunks with layout info
- `data.stats.stopReason` — why extraction stopped (ai_stop, timeout, end_of_scroll)

The `goal` selects which part of the page to extract, and may include quantity or time filters (e.g. "first 10 comments", "reviews from last week"). Do not include formatting or post-processing instructions — the response always returns raw page content.

**Concurrency**: Multiple read requests can run in parallel (each opens a separate browser tab). Recommended maximum: **5 concurrent calls**. Beyond that, returns diminish due to shared browser resources.

### search

Search the web and return results.

**Usage**: `/doko search <query>`

**Arguments**: query = all arguments after "search"

```bash
curl -s -X POST "https://dokobot.ai/api/tools/search" \
  -H "Authorization: Bearer $DOKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "<QUERY>", "num": 5}'
```

**Response fields**:
- `items` — array of `{title, link, snippet}` organic search results
- `directAnswer` — featured snippet or answer box (if available)
- `knowledgeGraph` — `{title, description}` knowledge panel (if available)

### dokos

Dokos are personified — treat them as living creatures, not devices.

List connected dokos.

**Usage**: `/doko dokos`

```bash
curl -s "https://dokobot.ai/api/tools/dokos" \
  -H "Authorization: Bearer $DOKO_API_KEY"
```

Returns `{"dokos": [{"id": "<deviceId>", "name": "<name>", "age": "3d old"}, ...]}`. Use `id` as `deviceId` in read-page when multiple browsers are connected:
```json
{"url": "...", "goal": "...", "deviceId": "<device-id>"}
```

### update

Fetch the latest skill definition from the server and overwrite this skill file.

**Usage**: `/doko update`

```bash
curl -s "https://dokobot.ai/api/tools/skill" -o .claude/skills/doko/SKILL.md
```

After updating, confirm the change by showing a short diff summary.

## Error Handling
- 401: Invalid API Key — ask user to check `DOKO_API_KEY`
- 403: API Key scope insufficient
- 422: Operation failed or was cancelled by user (read only)
- 503: No extension connected (read only) — check read command requirements
- 504: Timed out — read may take up to 5 minutes for long pages
