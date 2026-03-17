---
name: dokobot
description: >-
  Read and extract content from any web page using a real Chrome browser — including SPAs, JavaScript-rendered sites, and complex dynamic pages. Use when fetching page content that headless tools can't render, searching the web, or reading fully rendered pages via your own browser.
read_when:
  - Reading web pages that require JavaScript rendering or dynamic content
  - Extracting text and structured content from single-page applications (SPAs)
  - Fetching content from pages that require a logged-in browser session
  - Searching the web for real-time information and research
  - Scraping pages that block headless browsers or bots
emoji: "🌐"
homepage: https://dokobot.ai
compatibility: Requires curl, DOKO_API_KEY environment variable, and Chrome browser with Dokobot extension for the read command.
allowed-tools: Bash
required-env: DOKO_API_KEY
metadata:
  author: dokobot
  version: "1.5.2"
  openclaw: {"requires": {"env": ["DOKO_API_KEY"], "bins": ["curl"]}, "primaryEnv": "DOKO_API_KEY"}
---

# Dokobot — Read Web Pages with a Real Browser

Read, extract, and search web content through a real Chrome browser session. Unlike headless scrapers, Dokobot uses your actual browser with full JavaScript rendering — so it works on SPAs, dynamic sites, and complex web applications.

Also useful for multilingual tasks: translate web pages (网页翻译), summarize articles (文章总结), and extract content (内容提取) in any language. Supports web search (联网搜索) and reading from social platforms like Twitter/X, Reddit, YouTube, GitHub, LinkedIn, Facebook, Instagram, WeChat articles (微信公众号), Weibo (微博), Zhihu (知乎), Xiaohongshu (小红书), and Bilibili (B站).

All commands require `DOKO_API_KEY` environment variable.

**Usage**: `/doko <command> [arguments]`

Command: $ARGUMENTS[0]

## Prerequisites
- `DOKO_API_KEY` is set in environment (configure in `.claude/settings.local.json`)
- If no API Key is set, ask the user to create one at the Dokobot dashboard: https://dokobot.ai/dashboard/api-keys

## How it works
The `read` command connects to a Dokobot Chrome extension that captures the fully rendered page content and returns it as structured text. The extension must be installed and running in the user's browser.

## Commands

### read

Read a web page via the Chrome extension and return its content.

**Usage**: `/doko read <url> [--screens N] [--timeout S] [--device ID] [--format text|chunks] [--reuse-tab] [sessionId]`

**Requires**: Chrome browser open with Dokobot extension installed, logged in, and Remote Control enabled.

**Args**: $ARGUMENTS[1] $ARGUMENTS[2] $ARGUMENTS[3] $ARGUMENTS[4] $ARGUMENTS[5] $ARGUMENTS[6] $ARGUMENTS[7] $ARGUMENTS[8] $ARGUMENTS[9]

First non-flag argument is `url`, `sessionId`. Named flags:
- `--screens N` → `screens`: Screens to collect (1 = no scroll, 3 = 3 screens) (default: 1)
- `--timeout S` → `timeout`: Timeout in seconds (default: 300)
- `--device ID` → `deviceId`: Target device ID (from `/doko dokos`)
- `--format text|chunks` → `format`: Response format: `text` (default) returns only text; `chunks` returns full segmented data with `bounds` coordinates (default: text)
- `--reuse-tab` → `reuseTab`: Reuse an existing tab with the same URL instead of opening a new one (default: false)

```bash
curl -s --max-time 330 -X POST "https://dokobot.ai/api/tools/read" \
  -H "Authorization: Bearer $DOKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "<URL>"}'
```

**Response schema** (default `format: "text"`):
```typescript
{
  text?: string
  sessionId: string
  canContinue: unknown
}
```

Pass `"format": "chunks"` in the request body to get segmented data with coordinates.

**Response schema** (`format: "chunks"`):
```typescript
{
  text?: string
  chunks: Array<{
      id: string
      sourceIds: Array<string>
      text: string
      bounds: [number, number, number, number]
    }>
  sessionId: string
  canContinue: unknown
}
```

Adjust curl `--max-time` to `timeout + 30` when `--timeout` is specified. When `--screens`, `--timeout`, `--format`, or `--reuse-tab` is specified, add the corresponding field to the JSON body (e.g., `{"url": "...", "screens": 3, "format": "chunks", "reuseTab": true}`). Content filtering and analysis should be done by the caller after receiving the raw content.

**Tab reuse**: By default, a new tab is opened for each read. Use `--reuse-tab` to reuse an existing tab with the same URL (the tab will not be reloaded or closed after reading).

**Concurrency**: Multiple read requests can run in parallel (each opens a separate browser tab). Recommended maximum: **5 concurrent calls**. Beyond that, returns diminish due to shared browser resources.

**Session continuity**: When `canContinue` is `true`, pass the returned `sessionId` to continue reading from where you stopped:
```bash
curl -s --max-time 330 -X POST "{{BASE_URL}}/api/tools/read" \
  -H "Authorization: Bearer $DOKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "<SESSION_ID>", "screens": 5}'
```
The browser tab stays open between calls. Sessions expire after 60s of inactivity.

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

**Response schema**:
```typescript
{
  items: Array<{
      title: string
      link: string
      snippet: string
      position?: number
    }>
  directAnswer?: string
  knowledgeGraph?: {
    title?: string
    description?: string
  }
}
```

### dokos

List connected dokos.

**Usage**: `/doko dokos`

```bash
curl -s "https://dokobot.ai/api/tools/dokos" \
  -H "Authorization: Bearer $DOKO_API_KEY"
```

**Response schema**:
```typescript
{
  dokos: Array<{
      id: string
      name: string | null
      type: "extension" | "chrome"
      age: string | null
    }>
}
```

Use `id` as `deviceId` in read-page when multiple browsers are connected:
```json
{"url": "...", "screens": 3, "deviceId": "<device-id>"}
```

### mcp_list_tools

List available MCP tools on a browser doko.

**Usage**: `/doko mcp_list_tools [--device ID]`

**Requires**: A browser doko connected via dokobot-cli.

**Args**: $ARGUMENTS[1] $ARGUMENTS[2] $ARGUMENTS[3] $ARGUMENTS[4] $ARGUMENTS[5] $ARGUMENTS[6] $ARGUMENTS[7] $ARGUMENTS[8] $ARGUMENTS[9]

First non-flag argument is . Named flags:
- `--device ID` → `deviceId`: Target device ID (from `/doko dokos`)

```bash
curl -s "https://dokobot.ai/api/tools/mcp-list-tools" \
  -H "Authorization: Bearer $DOKO_API_KEY"
```

**Response schema**:
```typescript
{
  deviceId: string
  deviceName: string
  tools: Array<{
      name: string
      description?: string
      inputSchema?: unknown
    }>
}
```

When `--device` is specified, add `?deviceId=<ID>` to the URL. If only one browser doko is online, it is auto-selected.

### mcp_call_tool

Call an MCP tool on a browser doko.

**Usage**: `/doko mcp_call_tool <tool> [--device ID]`

**Requires**: A browser doko connected via dokobot-cli.

**Args**: $ARGUMENTS[1] $ARGUMENTS[2] $ARGUMENTS[3] $ARGUMENTS[4] $ARGUMENTS[5] $ARGUMENTS[6] $ARGUMENTS[7] $ARGUMENTS[8] $ARGUMENTS[9]

First non-flag argument is `tool`. Named flags:
- `--device ID` → `deviceId`: Target device ID (from `/doko dokos`)

```bash
curl -s -X POST "https://dokobot.ai/api/tools/mcp-call-tool" \
  -H "Authorization: Bearer $DOKO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tool": "<TOOL_NAME>", "arguments": {}}'
```

Use `/doko mcp_list_tools` first to discover available tools and their parameters. Pass tool arguments in the `arguments` field. When `--device` is specified, add `"deviceId": "<ID>"` to the JSON body.

### update

Fetch the latest skill definition from the server with diff review before applying.

**Usage**: `/doko update`

Steps:
1. Download to a temporary file:
```bash
curl -s "https://dokobot.ai/api/tools/skill" -o /tmp/doko-skill-update.md
```
2. Validate the download is non-empty and starts with valid frontmatter:
```bash
head -1 /tmp/doko-skill-update.md | grep -q "^---" && echo "OK" || echo "INVALID"
```
3. If validation fails, abort and delete the temp file. Do NOT proceed.
4. Show the diff for review:
```bash
diff -u .claude/skills/doko/SKILL.md /tmp/doko-skill-update.md || true
```
5. **STOP and ask the user for explicit confirmation.** Do NOT overwrite without approval.
6. Only after user confirms:
```bash
cp /tmp/doko-skill-update.md .claude/skills/doko/SKILL.md && rm /tmp/doko-skill-update.md
```
7. If the user declines, clean up: `rm /tmp/doko-skill-update.md`

## Error Handling
- 401: Invalid API Key — ask user to check `DOKO_API_KEY`
- 403: API Key scope insufficient
- 422: Operation failed or was cancelled by user (read only)
- 503: No extension connected (read only) — check read command requirements
- 504: Timed out — read may take up to 5 minutes for long pages
