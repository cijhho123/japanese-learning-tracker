# Japanese Learning Tracker

State repository for Claude Code scheduled routines that aggregate Japanese learning resources.

## Purpose

Manages state for Claude Code cloud routines that intelligently select and summarize Japanese learning articles from multiple sources. Tracks ingested articles across routine runs to prevent duplicate summaries and maintain learning continuity across different sources and learning progressions.

## How It Works

The Claude Code routine runs **3 times daily** (8:00, 12:00, 17:00 Israel time) and:

1. Fetches the main pages of all configured Japanese learning sources
2. Intelligently selects **one article per run** (or multiple short topics from a single source)
3. Understands prerequisite relationships between articles:
   - For sources with explicit hierarchy (Tatsumoto), respects stated order
   - For sequential sources (Imabi), tracks position and suggests coherent progression
   - For nested/mega-pages (ixrec), extracts individual topics and tracks at subtopic level
   - For independent topic sources (Tofugu, Yoku.bi), infers dependencies from content
4. Generates a summary with key points and a link to the full article
5. Sends to Telegram (or reads in claude.ai/code/scheduled)
6. Logs the article to `state.json` to avoid resending

## State File (`state.json`)

The state file tracks global configuration and per-source progress with this structure:

### Config Section

```json
{
  "config": {
    "source_rotation_order": ["tofugu", "yoku_bi", "tatsumoto", "ixrec", "imabi"],
    "current_rotation_index": 0,
    "run_schedule": null,
    "last_config_update": null
  }
}
```

**Fields:**
- `source_rotation_order`: Array defining the order sources are cycled through. Each routine run picks the next source in this list.
- `current_rotation_index`: Routine increments this after each run to track position. When it reaches the end of the array, it wraps back to 0.
- `run_schedule`: `null` for now. Can be customized in the future to assign specific sources to specific times (see [Future Customization](#future-customization) below).
- `last_config_update`: ISO timestamp of when config was last modified.

### Per-Source Structure

Each source has:

```json
{
  "source_key": {
    "url": "source URL",
    "description": "what this source covers",
    "progress_tracking": "how to understand progression (sequential, hierarchical, independent, etc.)",
    "used_articles": [],
    "last_updated": "ISO timestamp",
    "current_section": "optional — for hierarchical sources",
    "next_article_index": "optional — for sequential sources",
    "structure_notes": "reference notes about site structure and how to navigate it"
  }
}
```

## Source Details

### **Tofugu** (`tofugu`)
- **URL:** https://www.tofugu.com/
- **Type:** Independent topic-based lessons
- **Coverage:** Japanese language and culture with progressive difficulty
- **Structure:** Topics accessible via categories/archives; no required sequence
- **Strategy:** Select based on inferred prerequisites and topic dependencies
- **Notes:** Topics can be standalone or have implicit learning relationships; routine should understand when one topic builds on another

### **Yoku.bi** (`yoku_bi`)
- **URL:** https://yoku.bi/
- **Type:** Independent resource guides
- **Coverage:** Japanese language learning resources and guides
- **Structure:** TBD on first site visit; navigate carefully to understand topology
- **Strategy:** Select based on content relationships and logical progression
- **Notes:** Structure discovered during first routine run; routine should adapt to how site is organized

### **Tatsumoto** (`tatsumoto`)
- **URL:** https://tatsumoto.neocities.org/blog/table-of-contents
- **Type:** Hierarchical structured guide
- **Coverage:** Comprehensive Japanese learning with explicit prerequisites
- **Structure:** Table of contents provides explicit prerequisite chains and hierarchy
- **Strategy:** Follow TOC order; always respect stated prerequisites before serving an article
- **Notes:** This is the most structured source; use it as a backbone for learning coherence; track current section to maintain position

### **ixrec** (`ixrec`)
- **URL:** https://ixrec.neocities.org/
- **Type:** Nested mega-pages with subtopics
- **Coverage:** Comprehensive nested articles on Japanese grammar, particles, and related topics
- **Structure:** Large pages (e.g., `/all-about-particles`) contain many subtopics/sections; not meant to be consumed as full pages
- **Strategy:** Extract and track individual topics/sections **within pages**, not full pages
- **Example:** `/all-about-particles` should be broken into individual particle explanations; track each separately
- **Notes:** Critical to select subtopics intelligently; a single page may take weeks to fully digest if broken into sections

### **Imabi** (`imabi`)
- **URL:** https://imabi.org/
- **Type:** Sequential numbered lessons
- **Coverage:** Sequential numbered Japanese grammar lessons in progressive order
- **Structure:** Articles are numbered and ordered; follow sequence or select based on topics
- **Strategy:** Track `next_article_index` to maintain position; optionally suggest following sequence for coherent progression
- **Notes:** Natural ordering makes this good for structured learning; routine can suggest following sequence or allow user to jump

## Source Rotation

### Default Behavior (Current)

The routine cycles through sources in the order defined by `source_rotation_order`:

**Example cycle:**
- **8:00 AM:** Tofugu (index 0)
- **12:00 PM:** Yoku.bi (index 1)
- **5:00 PM:** Tatsumoto (index 2)
- **Next day 8:00 AM:** ixrec (index 3)
- **Next day 12:00 PM:** Imabi (index 4)
- **Next day 5:00 PM:** Back to Tofugu (index 0, wrapped)

After each run, the routine:
1. Increments `current_rotation_index`
2. Wraps back to 0 if the index exceeds array length
3. Commits the updated state.json to GitHub

This ensures you cycle through all sources evenly, one per run.

### Future Customization

When you want each time slot to have a dedicated source, modify `run_schedule`:

```json
{
  "config": {
    "source_rotation_order": ["tofugu", "yoku_bi", "tatsumoto", "ixrec", "imabi"],
    "current_rotation_index": 0,
    "run_schedule": {
      "08:00": "tatsumoto",
      "12:00": "imabi",
      "17:00": "ixrec"
    },
    "last_config_update": "2026-04-17T10:30:00Z"
  }
}
```

With this config, the routine will:
- Always use **Tatsumoto** at 8:00 AM
- Always use **Imabi** at 12:00 PM
- Always use **ixrec** at 5:00 PM

The `source_rotation_order` is then ignored in favor of `run_schedule`. When you want to switch back to rotation, set `run_schedule` to `null`.

## Tracking Strategy

### Independent Topics (Tofugu, Yoku.bi)
- No required order
- Routine infers dependencies from article content
- Marks articles as used to avoid repeats
- Suggests logical progression based on prerequisites discovered in content

### Hierarchical (Tatsumoto)
- Respects explicit prerequisite chains from TOC
- Maintains `current_section` to resume from where you left off
- Never serves an article before its stated prerequisites
- Enforces strict ordering per the guide's intended progression

### Sequential (Imabi)
- Tracks `next_article_index` to maintain position
- Can serve articles in order OR allow intelligent selection
- Use `next_article_index` to suggest "continue from here" vs "skip ahead"

### Nested/Mega-Pages (ixrec)
- Extracts individual topics from large pages
- Tracks at subtopic level, not page level
- Example: `/all-about-particles` becomes many tracked entries like `/all-about-particles#は`, `/all-about-particles#を`, etc.
- Prevents sending the same massive page twice by tracking specific sections

## `used_articles` Format

Each source's `used_articles` array contains objects with:
```json
{
  "title": "Article or topic title",
  "url": "Full URL or URL + anchor",
  "sent_at": "ISO timestamp when routine sent it",
  "source_key": "which source (for reference)",
  "subtopic": "optional — for nested pages, which subtopic was extracted"
}
```

## Routine Behavior

### Per Run
1. Read `state.json`
2. Determine which source to use:
   - If `run_schedule` is configured, use the mapping for current time
   - Otherwise, use `source_rotation_order[current_rotation_index]`
3. For the selected source:
   - Fetch the page
   - Understand available articles/topics
   - Check `used_articles` to avoid repeats
   - Intelligently pick one (or several short ones) based on:
     - Prerequisites already satisfied
     - Logical progression
     - What's already been sent
4. Generate summary with key points + full article link
5. Update state.json:
   - Add new article to `used_articles`
   - Increment `current_rotation_index` (if using rotation)
   - Update `last_updated` timestamp for the source
   - Commit and push to GitHub
6. Send summary to Telegram

### Deduplication
- Each run checks `used_articles` before suggesting anything
- If a source is exhausted (all articles used), routine should note this in feedback and consider:
  - Clearing `used_articles` to allow repeats
  - Skipping to next source in rotation
  - Asking user for guidance

### Prerequisites & Ordering
- **Tatsumoto:** Strict; always check prerequisite chain before suggesting
- **Imabi:** Suggest sequential; track `next_article_index` for resume-from-here logic
- **ixrec:** Extract subtopics smartly; track at subtopic level
- **Tofugu/Yoku.bi:** Infer from content; routine reasons about what should come next

## Example Run Sequence

**Day 1, 8:00 AM:**
- `current_rotation_index` = 0 → Tofugu
- Selects an article from Tofugu
- Sends summary + link
- Updates `used_articles` in tofugu section
- Increments `current_rotation_index` to 1
- Commits state.json

**Day 1, 12:00 PM:**
- `current_rotation_index` = 1 → Yoku.bi
- Selects an article from Yoku.bi
- Sends summary + link
- Updates state.json
- Increments `current_rotation_index` to 2
- Commits state.json

**Day 1, 5:00 PM:**
- `current_rotation_index` = 2 → Tatsumoto
- Checks Tatsumoto prerequisites
- Selects next unsatisfied prerequisite
- Sends summary + link
- Updates state.json
- Increments `current_rotation_index` to 3
- Commits state.json

**Day 2, 8:00 AM:**
- `current_rotation_index` = 3 → ixrec
- Extracts a subtopic from ixrec
- Sends summary + link
- Updates state.json
- Increments `current_rotation_index` to 4
- Commits state.json

**Day 2, 12:00 PM:**
- `current_rotation_index` = 4 → Imabi
- Uses `next_article_index` to suggest next sequential article
- Sends summary + link
- Updates state.json and increments `next_article_index`
- Wraps `current_rotation_index` back to 0 (cycle complete)
- Commits state.json

## Maintenance

- Review `used_articles` periodically to see learning progress across all sources
- If you want to reset a source's history, clear its `used_articles` array
- If you want to skip a source temporarily, remove it from `source_rotation_order` or add it back when ready
- Update `structure_notes` if site structure changes significantly
- Add new sources by extending the JSON with the same per-source structure
- Modify `run_schedule` to customize time-based source assignments

## Integration with Claude Code

This repo is cloned at routine startup and state is committed back after each run. The routine has full read/write access via your authenticated GitHub connection.

All changes are auto-committed with messages like:
```
Update state: sent [article_title] from [source_key]
```

This creates a clear audit trail of what was sent when.