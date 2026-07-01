# Feature: Inline Content Output for Large Scraping Responses

## Problem

When `mcp_scrapling_get` (and related tools) return large responses, they write the result to a JSON file on disk instead of returning it inline. This forces every consumer to perform an intermediate step:

1. Read the file path from the tool result
2. Execute a Node.js one-liner (or equivalent) to parse the JSON wrapper
3. Extract the actual content from the nested structure (`data.content[0]`)
4. Truncate to a readable length

This adds friction, extra tool calls, and cognitive overhead — especially during multi-source research sessions where 3+ URLs are scraped in parallel.

## Current Behavior

```
Tool returns:
  "Large tool result (39KB) written to file. Use the read_file tool to access
   the content at: c:\Users\...\content.json"

Consumer must then:
  node -e "const fs = require('fs'); const data = JSON.parse(fs.readFileSync('path/content.json', 'utf8')); console.log(data.content[0].substring(0, 6000));"
```

## Proposed Solution

Add two new parameters to scraping tools:

### 1. `returnInline` (boolean, default: `false`)

When `true`, return the extracted content directly in the tool response instead of writing to a file. For backward compatibility, defaults to `false` so existing behavior is preserved.

```json
{
  "url": "https://example.com/article",
  "extraction_type": "markdown",
  "returnInline": true
}
```

**Response:**
```
"## Article Title\n\nFull markdown content here...\n\n..."
```

### 2. `writeRawFile` (boolean, default: `false`)

When `true` and the response is still too large for inline return, write the extracted content as a **plain text / markdown file** instead of a JSON wrapper. This eliminates the need for JSON parsing — the file can be read directly with `read_file`.

```json
{
  "url": "https://example.com/article",
  "extraction_type": "markdown",
  "writeRawFile": true
}
```

**Response:**
```
"Content written to: c:\Users\...\content.md"
```

File contains raw markdown/text — no JSON wrapper.

## Design Rationale

| Approach | Pros | Cons |
|---|---|---|
| `returnInline: true` | Zero intermediate steps, simplest UX | May hit response size limits for very large pages |
| `writeRawFile: true` | No JSON parsing needed, file is directly readable | Still requires a file read, but it's a simple `read_file` not a Node.js parse |

Both can coexist: use `returnInline` for most cases, and `writeRawFile` as a fallback when content exceeds inline limits.

## Implementation Notes

- The existing JSON schema file (`schema.json`) written alongside `content.json` can be deprecated for the raw file path, or kept as optional metadata
- A size threshold (e.g., 10KB) could auto-switch from inline to file output without the user needing to specify both flags
- File extension should match extraction type: `.md` for markdown, `.txt` for text, `.html` for HTML

## Impact

- **Reduces tool calls per research session** by 1–3 (one Node.js parse per scraped URL eliminated)
- **Removes dependency on Node.js** being available in the terminal
- **Simplifies the scraping workflow** from a 2-step process to a 1-step process
- **No breaking changes** — both parameters are opt-in with safe defaults
