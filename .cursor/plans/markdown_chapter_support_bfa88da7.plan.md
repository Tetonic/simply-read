---
name: Markdown chapter support
overview: Add support for `.md` (markdown) files alongside existing `.docx` files as chapter input in the TTS app. The server will detect the file type by extension and parse accordingly.
todos:
  - id: list-files
    content: Rename listDocxFiles to listChapterFiles, broaden filter to .docx and .md
    status: pending
  - id: md-parser
    content: Add parseMarkdownToText() and parseChapterToText() dispatcher in server.js
    status: pending
  - id: base-name
    content: Fix hardcoded .docx extension strip in /api/generate to use path.extname()
    status: pending
  - id: frontend-msg
    content: Update empty-state status message in public/script.js
    status: pending
isProject: false
---

# Markdown Chapter File Support

Currently the app only reads `.docx` files from the `chapters/` directory using the `mammoth` library. This plan adds `.md` file support so either format can be used.

## Changes

### 1. Server: rename and broaden file listing

In [server.js](server.js), rename `listDocxFiles()` to `listChapterFiles()` and update the filter to accept both `.docx` and `.md` extensions:

```javascript
function listChapterFiles() {
  if (!fs.existsSync(chaptersDir)) return [];
  return fs
    .readdirSync(chaptersDir)
    .filter((file) => {
      const ext = file.toLowerCase();
      return ext.endsWith(".docx") || ext.endsWith(".md");
    });
}
```

Update the single call site at the `/api/chapters` endpoint to use the new name.

### 2. Server: add markdown-to-text parser

Add a `parseMarkdownToText()` function in [server.js](server.js) that reads the `.md` file and strips markdown syntax to produce clean plain text for TTS. No new dependency needed -- a lightweight regex-based strip handles headers, bold/italic, links, images, code fences, and horizontal rules:

```javascript
function parseMarkdownToText(filePath) {
  let text = fs.readFileSync(filePath, "utf8");
  text = text.replace(/^```[\s\S]*?^```/gm, "");       // fenced code blocks
  text = text.replace(/`([^`]+)`/g, "$1");               // inline code
  text = text.replace(/!\[([^\]]*)\]\([^)]+\)/g, "$1");  // images
  text = text.replace(/\[([^\]]+)\]\([^)]+\)/g, "$1");   // links
  text = text.replace(/^#{1,6}\s+/gm, "");               // headings
  text = text.replace(/(\*\*|__)(.*?)\1/g, "$2");         // bold
  text = text.replace(/(\*|_)(.*?)\1/g, "$2");            // italic
  text = text.replace(/^[-*]{3,}\s*$/gm, "");             // horizontal rules
  text = text.replace(/^[>\s]+/gm, (m) => m.replace(/>/g, "")); // blockquotes
  text = text.replace(/^\s*[-*+]\s+/gm, "");             // unordered list markers
  text = text.replace(/^\s*\d+\.\s+/gm, "");             // ordered list markers
  return text;
}
```

### 3. Server: unify text extraction by file type

Create a `parseChapterToText()` dispatcher that picks the right parser based on extension. Replace all calls to `parseDocxToText()` (in `/api/chapter-text` and `/api/generate`) with this:

```javascript
async function parseChapterToText(filePath) {
  if (filePath.toLowerCase().endsWith(".md")) {
    return parseMarkdownToText(filePath);
  }
  return parseDocxToText(filePath);
}
```

### 4. Server: fix base-name extraction in `/api/generate`

The audio filename currently hardcodes stripping `.docx`:

```javascript
const base = sanitizeBaseName(path.basename(chapterName, ".docx"));
```

Replace with `path.extname()` so it works for any extension:

```javascript
const ext = path.extname(chapterName);
const base = sanitizeBaseName(path.basename(chapterName, ext));
```

### 5. Frontend: update empty-state message

In [public/script.js](public/script.js), line 42, change the status hint from:

> `"Add .docx files to the chapters folder."`

to:

> `"Add .docx or .md files to the chapters folder."`

## Files changed

- `server.js` -- all server-side logic (steps 1-4)
- `public/script.js` -- one status string (step 5)

## No new dependencies

Markdown stripping is done with a small set of regex replacements, avoiding the need for an additional npm package.
