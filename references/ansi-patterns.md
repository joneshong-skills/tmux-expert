# ANSI SGR Parsing Patterns

## Quick Strip (Lossy)

For cases where color doesn't matter, strip all CSI sequences:

```javascript
// Strip ALL CSI sequences (safe, comprehensive)
text.replace(/\x1b\[[0-9;]*[A-Za-z]/g, '')

// WRONG — only matches 'm' terminator, misses cursor moves etc.
text.replace(/\x1b\[[0-9;]*m/g, '')
```

## Full SGR Parser (Lossless → HTML)

Parse `\x1b[` + semicolon-separated params + `m` terminator into CSS spans.

### Parameter Reference

| Code | Meaning | CSS |
|------|---------|-----|
| 0 | Reset all | Close all spans |
| 1 | Bold | `font-weight:bold` |
| 2 | Dim | `opacity:0.7` |
| 3 | Italic | `font-style:italic` |
| 4 | Underline | `text-decoration:underline` |
| 30-37 | FG standard colors | `color: PALETTE[n-30]` |
| 38;5;N | FG 256-color | `color: color256(N)` |
| 38;2;R;G;B | FG RGB | `color: rgb(R,G,B)` |
| 40-47 | BG standard colors | `background: PALETTE[n-40]` |
| 48;5;N | BG 256-color | `background: color256(N)` |
| 48;2;R;G;B | BG RGB | `background: rgb(R,G,B)` |
| 39 | FG default | Remove color |
| 49 | BG default | Remove background |
| 90-97 | FG bright colors | `color: BRIGHT_PALETTE[n-90]` |
| 100-107 | BG bright colors | `background: BRIGHT_PALETTE[n-100]` |

### 256-Color Palette Ranges

| Range | Meaning |
|-------|---------|
| 0-7 | Standard colors (same as 30-37) |
| 8-15 | Bright colors (same as 90-97) |
| 16-231 | 6x6x6 RGB cube: `16 + 36*r + 6*g + b` (r,g,b: 0-5) |
| 232-255 | Grayscale: `8 + 10*(N-232)` for each channel |

### Parser State Machine

```javascript
function parseAnsi(raw) {
  let html = '', fg = '', bg = '', bold = false, dim = false, italic = false, underline = false;
  let i = 0;

  while (i < raw.length) {
    if (raw[i] === '\x1b' && raw[i+1] === '[') {
      // Extract params until 'm'
      let j = i + 2, params = '';
      while (j < raw.length && raw[j] !== 'm' && /[0-9;]/.test(raw[j])) {
        params += raw[j]; j++;
      }
      if (raw[j] === 'm') {
        const codes = params ? params.split(';').map(Number) : [0];
        // Process codes (handle 38;5;N and 38;2;R;G;B sequences)
        applyStyles(codes);  // updates fg, bg, bold, etc.
        i = j + 1;
        continue;
      }
      // Non-SGR CSI — skip
      while (j < raw.length && !/[A-Za-z]/.test(raw[j])) j++;
      i = j + 1;
      continue;
    }
    // Regular character — emit with current style
    html += styledChar(raw[i], fg, bg, bold, dim, italic, underline);
    i++;
  }
  return html;
}
```

### Key Implementation Notes

1. **38/48 consume extra params**: When encountering code 38 or 48, look ahead:
   - Next code is 5 → consume 1 more (256-color index)
   - Next code is 2 → consume 3 more (R, G, B)

2. **Reset (code 0)** clears ALL attributes, not just the most recent one.

3. **Multiple codes in one sequence**: `\033[1;38;5;82m` = bold + FG color 82.
   Process left-to-right, consuming sub-params for 38/48.

4. **Empty params** (`\033[m`) is equivalent to `\033[0m` (full reset).

5. **Batch span generation**: Don't open/close a `<span>` per character.
   Track state changes and only emit new spans when style actually changes.
