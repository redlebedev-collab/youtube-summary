# YouTube Playlist Summary Extraction — AI Agent Protocol

## Goal

Extract structured Markdown summaries (конспекты) from YouTube playlist videos using Claude in Chrome. For each video: open the transcript panel, sample the auto-generated transcript, write a structured notes file. Produce an index file at the end.

---

## Autonomy Policy

**Work fully autonomously.** Once the user gives a playlist URL and target range (e.g. "videos 6–35"), process all videos end-to-end without interrupting. Only pause if:

- YouTube requires re-login.
- A video has no transcript and no fallback is possible.
- A tool call fails 2–3 times in a row.

Everything else — navigating videos, opening transcripts, extracting text, writing files, building the index — do silently and report only when all videos are done.

---

## Prerequisites

- Chrome browser open with Claude in Chrome extension connected.
- User logged into YouTube (for age-restricted or private videos).
- Target output folder created (e.g. `C:\Projects\Миллионы 2.0`).
- Cowork file access granted to the output folder.

---

## Step 1: Get All Video IDs from Playlist

1. Navigate to the playlist URL in the Chrome tab.
2. Run JavaScript to extract video IDs, indices, and titles:

```javascript
const items = document.querySelectorAll('ytd-playlist-video-renderer');
const result = [];
items.forEach(item => {
  const link = item.querySelector('a#video-title');
  if (!link) return;
  const title = link.textContent.trim();
  const href = link.getAttribute('href') || '';
  const m = href.match(/v=([^&]+)/);
  const idxM = href.match(/index=(\d+)/);
  const videoId = m ? m[1] : '';
  const index = idxM ? parseInt(idxM[1]) : null;
  if (index >= START && index <= END) {
    result.push(index + '|' + videoId + '|' + title);
  }
});
result.join('\n');
```

3. Save the list (index, videoId, title) for all subsequent steps.

> **Note:** If the playlist has 50+ videos and not all render, scroll the page first.

---

## Step 2: For Each Video — Open Transcript

```javascript
// Step A: Navigate to https://www.youtube.com/watch?v=VIDEO_ID

// Step B: Open "More actions" menu
document.querySelector('button[aria-label="Другие действия"]')?.click();

// Step C: (separate call) Click "Show transcript"
document.querySelector('button[aria-label="Показать текст видео"]')?.click();
```

> **Critical:** Steps B and C must be separate tool calls. The menu button renders asynchronously. If Step C returns empty panel (LEN:0), wait ~2 seconds and retry.

---

## Step 3: Extract Transcript Samples

```javascript
const p = document.querySelector('ytd-engagement-panel-section-list-renderer');
if (!p) { 'NO_PANEL'; } else {
  const t = p.innerText.split('\n').filter(l =>
    l.trim() &&
    !l.match(/^\d+:\d+$/) &&
    !l.match(/^\d+ (секунд|минут|час)/) &&
    !l.match(/^Расш/) &&
    !l.match(/^Поис/)
  ).join(' ');
  window._transcript = t;
  const L = t.length;
  if (L < 50) { 'SHORT:' + p.innerText.substring(0, 300); }
  else {
    'LEN:' + L + '\n' +
    [0, 0.2, 0.4, 0.6, 0.8].map((q, i) =>
      'S' + (i+1) + ':' + t.substring(Math.floor(L*q), Math.floor(L*q) + 200)
    ).join('\n');
  }
}
```

Returns `LEN:XXXXX` and `S1`–`S5`: 200-char samples from 0%, 20%, 40%, 60%, 80% of transcript.

> **Why samples?** Transcripts can be 30K–170K chars. JS display truncates at ~1500 chars. Five 200-char samples fit in one call and cover the full arc of the video.

---

## Step 4: Write the Markdown File

```bash
python3 << 'PYEOF'
folder = "/path/to/output"
content = """# VIDEO TITLE

**Плейлист:** PLAYLIST NAME
**Позиция в плейлисте:** INDEX
**URL:** https://www.youtube.com/watch?v=VIDEO_ID
**Длительность:** ~DURATION

---

## Конспект

NARRATIVE SUMMARY

### Ключевые тезисы

- **POINT 1** — detail
- **POINT 2** — detail

---

## Фрагменты транскрипта

**Начало:** S1 TEXT
**1/4:** S2 TEXT
**1/2:** S3 TEXT
**3/4:** S4 TEXT
**Конец:** S5 TEXT

---

## Примечания

- Объём транскрипта: LEN символов
"""
with open(folder + "/NN-FILENAME.md", "w", encoding="utf-8") as f:
    f.write(content)
PYEOF
```

**File naming:** `NN-Короткое-название.md` where NN = playlist index (zero-padded).

---

## Step 5: Create the Index File

After all videos are processed, write `00-Индекс-курса.md`:

```markdown
# PLAYLIST NAME — Индекс

| # | Позиция | Название | Длит. | Краткое содержание | Файл |
|---|---------|----------|-------|--------------------|------|
| 1 | 6 | Урок 1 | ~30 мин | Что разбирается | [06](06-filename.md) |
```

---

## Handling Edge Cases

| Situation | Action |
|-----------|--------|
| Panel loads empty (LEN:0) after click | Wait ~2s and re-run extraction JS |
| No transcript button found | Create minimal file noting "No transcript" |
| Batch call returns EMPTY on 2nd JS | Menu click and transcript load need separate calls |
| Very short transcript (<500 chars) | Include full text verbatim |

---

## Technical Notes

### Why not fetch transcript via API?
YouTube's transcript `baseUrl` contains session tokens that expire. `fetch()` in-page returns empty. The DOM panel approach is reliable.

### Why 5 samples of 200 chars?
JS tool display truncates at ~1500 chars. Asking for `substring(0, 10000)` only shows ~1500. Five 200-char samples from evenly spaced positions cover beginning, middle, and end — sufficient for accurate summaries.

### Transcript filter regex
Remove from panel text:
- `^\d+:\d+$` — MM:SS timestamp
- `^\d+ (секунд|минут|час)` — spelled-out timestamp
- `^Расш` — "Расшифровка видео" heading
- `^Поис` — "Поиск в расшифровке" label

### Why Python for file writing?
Bash heredoc breaks on Cyrillic and special chars. Python with `encoding="utf-8"` always works.

---

## Quality Checklist

- [ ] All target videos processed (no gaps)
- [ ] Each file has: metadata, конспект, ключевые тезисы, фрагменты транскрипта
- [ ] Transcript length noted in примечания
- [ ] Index file (00-...) created with complete table
- [ ] File names follow NN-Short-Name.md convention
