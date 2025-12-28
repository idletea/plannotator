# Plannotator

A plan review UI for Claude Code that intercepts `ExitPlanMode` and lets you approve or request changes with annotated feedback.

## How It Works

### Hook Flow

```
Claude calls ExitPlanMode
        │
        ▼
PermissionRequest hook fires
        │
        ▼
hooks/exit-plan-mode.sh reads plan from stdin JSON
        │
        ▼
Bun server starts (random port)
        │
        ▼
Browser opens to Plannotator UI
        │
        ▼
User reviews plan, optionally adds annotations
        │
        ├─── Approve ───► JSON output: {"decision": {"behavior": "allow"}}
        │                       │
        │                       ▼
        │                 Claude proceeds with implementation
        │
        └─── Request Changes ───► JSON output: {"decision": {"behavior": "deny", "message": "..."}}
                                       │
                                       ▼
                                 Claude receives feedback, revises plan
```

### Key Files

| File | Purpose |
|------|---------|
| `hooks/exit-plan-mode.sh` | Shell script invoked by Claude Code hook system |
| `server/index.ts` | Bun server that serves UI and handles approve/deny API |
| `App.tsx` | React app with API mode detection |
| `dist/index.html` | Built single-file app served by Bun |

### Hook Configuration

The hook is configured in `.claude/settings.local.json`:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "matcher": "ExitPlanMode",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/planning-hook/hooks/exit-plan-mode.sh",
            "timeout": 600
          }
        ]
      }
    ]
  }
}
```

**Why PermissionRequest instead of PreToolUse?**

- `PermissionRequest` fires when Claude Code is about to show a permission dialog
- Hook can return JSON to `allow` or `deny` directly — no second dialog
- `PreToolUse` would require a second approval step in Claude Code UI

### Server API

The Bun server exposes three endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/plan` | GET | Returns plan content as JSON |
| `/api/approve` | POST | User approved the plan |
| `/api/deny` | POST | User denied with feedback in request body |

### Multi-Instance Support

Each Plannotator instance runs on a random port (`Bun.serve({ port: 0 })`). The browser's same-origin policy ensures each tab talks to its own server — no cross-instance interference.

### Hook Output Format

The server outputs JSON to stdout for the PermissionRequest hook:

**Approve:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow"
    }
  }
}
```

**Deny:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "deny",
      "message": "# Plan Feedback\n\n..."
    }
  }
}
```

## Annotation Data Model

### Types (`types.ts`)

```typescript
enum AnnotationType {
  DELETION = 'DELETION',      // Remove this text
  INSERTION = 'INSERTION',    // Add new text (originalText is context)
  REPLACEMENT = 'REPLACEMENT', // Replace originalText with text
  COMMENT = 'COMMENT',        // Comment on originalText
}

interface Annotation {
  id: string;              // From web-highlighter or `codeblock-{timestamp}`
  blockId: string;         // References Block.id (legacy, for sorting)
  startOffset: number;     // Char offset in block (legacy)
  endOffset: number;       // End char offset (legacy)
  type: AnnotationType;
  text?: string;           // User text for comment/replacement/insertion
  originalText: string;    // The selected/highlighted text
  createdA: number;        // Timestamp for ordering
  startMeta?: { parentTagName, parentIndex, textOffset }; // web-highlighter DOM pos
  endMeta?: { parentTagName, parentIndex, textOffset };
}

interface Block {
  id: string;              // e.g., "block-0"
  type: 'paragraph' | 'heading' | 'blockquote' | 'list-item' | 'code' | 'hr';
  content: string;
  level?: number;          // For headings (1-6)
  order: number;
  startLine: number;       // 1-based line in original markdown
}
```

### Annotation Creation Flow

1. User selects text → `web-highlighter` fires `CREATE` event
2. In **selection mode**: Toolbar appears with type options
3. In **redline mode**: Auto-creates `DELETION` annotation
4. `createAnnotationFromSource()` builds `Annotation` object
5. Annotation stored in `App.tsx` state: `useState<Annotation[]>([])`

### Code Block Annotations

Code blocks can't use web-highlighter (content not directly selectable). Instead:
- Hover triggers `CodeBlockToolbar`
- Manual `<mark>` wrapper created with `range.surroundContents()`
- ID format: `codeblock-{timestamp}`
- Must handle removal separately via `data-bind-id` attribute

### Serialization

**Current**: `exportDiff()` in `utils/parser.ts` — lossy human-readable export:
```markdown
## 1. Remove this
\`\`\`
selected text here
\`\`\`

## 2. Change this
**From:** \`\`\`old\`\`\`
**To:** \`\`\`new\`\`\`
```

**No round-trip persistence yet**. Considerations:
- `startMeta`/`endMeta` fragile if DOM changes
- `blockId` + offsets more stable but still break on content edits
- Options: localStorage, server-side JSON, adjacent `.annotations.json`

---

## Portable Sharing Design (textarea.my Pattern)

Inspired by [textarea.my](https://github.com/antonmedv/textarea) which stores everything in URL hash.

### Core Mechanism

```javascript
// Compress: string → deflate → base64url
async function compress(string) {
  const byteArray = new TextEncoder().encode(string);
  const stream = new CompressionStream('deflate-raw');
  const writer = stream.writable.getWriter();
  writer.write(byteArray);
  writer.close();
  const buffer = await new Response(stream.readable).arrayBuffer();
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}

// Decompress: base64url → inflate → string
async function decompress(b64) {
  const binary = atob(b64.replace(/-/g, '+').replace(/_/g, '/'));
  const byteArray = Uint8Array.from(binary, c => c.charCodeAt(0));
  const stream = new DecompressionStream('deflate-raw');
  const writer = stream.writable.getWriter();
  writer.write(byteArray);
  writer.close();
  const buffer = await new Response(stream.readable).arrayBuffer();
  return new TextDecoder().decode(buffer);
}
```

### Minimal Shareable Format

Strip annotations to essential data only:

```typescript
// Full Annotation → Shareable tuple
// [type_char, originalText, text?]
type ShareableAnnotation =
  | ['D', string]           // Deletion
  | ['R', string, string]   // Replacement: original, replacement
  | ['C', string, string]   // Comment: original, comment
  | ['I', string, string];  // Insertion: context, new text

// Example transformation
const shareable = annotations.map(a => {
  const t = a.type[0] as 'D'|'R'|'C'|'I';
  return a.text ? [t, a.originalText, a.text] : [t, a.originalText];
});
```

### Compression Results (Tested)

| Annotations | Raw JSON | Compressed | URL Safe? |
|-------------|----------|------------|-----------|
| 5           | 351 chars | 316 chars | ✅ Yes |
| 100         | ~7KB     | 383 chars | ✅ Yes |

Deflate excels at repetitive text. URL limit: ~2KB safe, ~64KB modern browsers.

### Sharing Approaches

**Option A: Annotations Only (Same Plan)**
```
plannotator.app/#<compressed-annotations>
```
- Assumes recipients have identical plan
- Smallest URL, ideal for team collaboration

**Option B: Plan Hash + Annotations**
```
plannotator.app/#<plan-hash>.<compressed-annotations>
```
- Include first 8 chars of SHA-256 of plan content
- Warn if plan doesn't match: "These annotations were made on a different version"

**Option C: Full Plan + Annotations**
```
plannotator.app/#<compressed-plan-and-annotations>
```
- Self-contained, works without server
- Larger URL but still feasible for most plans

### Reconstruction on Load

```typescript
async function loadFromHash(hash: string): Promise<Annotation[]> {
  const data = JSON.parse(await decompress(hash));
  return data.map(([type, original, text], i) => ({
    id: `shared-${i}`,
    blockId: '',  // Reconstruct by text search
    startOffset: 0,
    endOffset: 0,
    type: { D: 'DELETION', R: 'REPLACEMENT', C: 'COMMENT', I: 'INSERTION' }[type],
    text,
    originalText: original,
    createdA: Date.now() + i, // Preserve order
    // startMeta/endMeta: Reconstruct when highlighting
  }));
}

// Find position by text matching
function findTextPosition(blocks: Block[], originalText: string) {
  for (const block of blocks) {
    const idx = block.content.indexOf(originalText);
    if (idx !== -1) {
      return { blockId: block.id, startOffset: idx, endOffset: idx + originalText.length };
    }
  }
  return null; // Orphaned annotation
}
```

### Collaborative Flow

```
Developer A                          Developer B
    │                                     │
    ├── Reviews plan                      │
    ├── Adds annotations                  │
    ├── Clicks "Share"                    │
    │   └── Copies URL with #hash         │
    │                                     │
    ├───────── Sends URL ─────────────────►
    │                                     │
    │                                     ├── Opens URL
    │                                     ├── Annotations auto-load
    │                                     ├── Adds more annotations
    │                                     ├── Clicks "Share"
    │                                     │   └── New URL with merged annotations
    │                                     │
    ◄───────── Sends URL ─────────────────┤
    │                                     │
```

### Edge Cases

1. **Text not found**: Show as "orphaned" annotation in sidebar
2. **Duplicate matches**: Use first match, or require unique context
3. **Plan changed**: Hash mismatch warning, attempt best-effort matching
4. **URL too long**: Offer download as `.annotations.json` file instead

### Key Insight from textarea.my/md

The `md.html` variant stores **raw markdown** but renders styled output:
- Storage: `article.textContent` (raw markdown)
- Display: `parseMarkdown()` wraps syntax in `<span class="md-*">`
- Editing: `contenteditable="plaintext-only"` captures raw text

**Implication for Plannotator**: Annotations should reference **text content**, not DOM.
The DOM is ephemeral (rebuilt on each load). Text matching is stable.

```
Load Flow:
  hash → decompress → { plan: string, annotations: [] }
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
    parseMarkdown(plan)              for each annotation:
         → DOM                         find text in DOM
                                       apply <mark> highlight
```

## Development

```bash
# Install dependencies
npm install

# Run dev server (for UI development)
npm run dev

# Build for production
npm run build

# Test server standalone
bun run server/index.ts "# Test Plan\n\nSome content"
```

## Requirements

- Bun runtime
- Claude Code with hooks support
- macOS (uses `open` command for browser)
