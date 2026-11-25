# Tiered Loading System — Phase 1 Detailed Procedures

## Context Budget

| Resource | Token Estimate | Practical Limit |
|----------|---------------|-----------------|
| Meeting index entry | ~50 tokens | 500+ meetings |
| Meeting summary | ~500 tokens | 50-100 meetings |
| Full transcript (60 min) | ~12,000-16,000 tokens | 8-12 meetings |
| Available context budget | ~150,000 tokens | After system/tools overhead |

**Strategy**: Cast a wide net at the index level, progressively load detail.

## Section 0: Pick the Access Route (read first)

### Step 0.1: Detect the meeting access tier

Make one cheap call with the configured provider (for example `search_meetings(query="test", limit=1)`):

- **Cloud MCP responds** (for Granola: the official Granola cloud MCP) → Tier 1. Use Steps 1.2-1.5 as written.
- **Local MCP responds** (for Granola: the community local-cache MCP) → Tier 2. Same steps, but search may be keyword-only, so use more query variations.
- **Neither responds, `archive_dir` is set** → Tier 3. Use "Archive-direct loading" below.
- **Nothing else available, `notes_dir` is set** → Tier 4. Treat it as the `manual` provider.

Granola note: since Granola v7.277 (May 2026) the desktop app encrypts its local cache, so community MCPs that read the cache file get no data. Use the cloud MCP or an archive instead (see `docs/providers.md`).

### Step 0.2: Check caches and the daily briefing first

Run SKILL.md Phase 1 pre-steps (extraction cache, existing artifacts, synthesis cache, daily briefing) before any meeting call. A fresh cache or today's briefing often makes meeting loading unnecessary.

### Archive-direct loading (Tier 3)

Use when no meeting MCP is available in this runtime but meetings have been exported to `providers.meetings.config.archive_dir` (for example by a scheduled export job).

1. **Index:** if `{archive_dir}/index.json` exists, read it and filter by date, then by title keywords, then prefer entries with a transcript or notes. Otherwise list the files and filter by name and modified date.
2. **Notes:** read each meeting file. Use the first non-empty field among: rendered markdown notes, plain-text notes, structured notes (walk the content tree and collect the text), AI summary panels.
3. **Transcripts:** read the matching transcript file for the top-ranked meetings only, as in Step 1.4.
4. **Freshness:** say how current the archive is (from its sync log or newest entry). A meeting from earlier today may not be exported yet.

## Step 1.1: Parse Arguments

Extract from user input:
- Topic/query string
- Folder filter (optional)
- Video URLs (optional)
- Depth preference: `--deep` (fewer meetings, full transcripts) or `--broad` (many meetings, summaries)

## Step 1.2: Tier 1 — Discovery Search (Unlimited)

Cast the widest possible net using the configured meeting provider.

### Meeting Discovery Dispatch

| If `providers.meetings.type` = | Discovery Call |
|--------------------------------|----------------|
| `granola` | `search_meetings(query="[topic]", limit=100)` via Granola MCP |
| `otter` | `search_transcripts(query="[topic]", limit=100)` via Otter MCP |
| `fireflies` | `search_meetings(query="[topic]", limit=100)` via Fireflies MCP |
| `google-meet` | Search Google Calendar API for events matching topic, then check Google Drive for auto-transcripts |
| `notion` | `notion-search(query="[topic]")` scoped to `config.meetings.config.notion_meeting_db` |
| `manual` | `Glob("**/*.md", path=config.meetings.config.notes_dir)` then `Grep(pattern="[topic]")` over matches |

For broad topics, run supplementary searches with variations:
- Original query + synonyms
- Key participant names + topic
- Related feature names + topic

**Deduplicate** results by meeting ID / filename before presenting.

**Present discovery results as index table:**
```
## Discovery: Found [N] meetings related to "[topic]"

| # | Meeting | Date | Participants | Duration | Relevance |
|---|---------|------|--------------|----------|-----------|
| 1 | [Title] | [Date] | [Names] | [Min] | High |
...

**Context budget**: Loading all [N] transcripts would use ~[X] tokens.
**Recommended**: Summaries for top [Y], full transcripts for top 5-8.
```

## Step 1.3: Tier 2 — Summary Loading (Broad Context)

### Summary Dispatch

| If `providers.meetings.type` = | Summary Call |
|--------------------------------|--------------|
| `granola` | `get_meeting_details(meeting_id="[id]")` |
| `otter` | `get_transcript_summary(transcript_id="[id]")` |
| `fireflies` | `get_meeting_summary(meeting_id="[id]")` |
| `google-meet` | Read first 500 words of Drive transcript |
| `notion` | Read page properties + first content block |
| `manual` | Read first 30 lines of each matching file |

Build working context: key decisions, participants, action items, topic keywords.

**Token budget for Tier 2**: ~15,000-25,000 tokens (30-50 summaries)

**In `--deep` mode**: Reduce to 10-15 summaries (~5-10K tokens) to allocate more budget to Tier 3.

## Step 1.4: Tier 3 — Deep Dive Loading (Full Transcripts)

### Transcript Dispatch

| If `providers.meetings.type` = | Transcript Call |
|--------------------------------|-----------------|
| `granola` | `get_meeting_transcript(meeting_id="[id]")` |
| `otter` | `get_full_transcript(transcript_id="[id]")` |
| `fireflies` | `get_transcript(meeting_id="[id]")` |
| `google-meet` | Read full Drive transcript document |
| `notion` | Read full page content |
| `manual` | Read entire file |

**Selection criteria** (priority order):
1. Highest relevance score from search
2. Most recent meetings (recency bias)
3. Meetings with key stakeholders
4. Meetings explicitly requested by user

**Token budget for Tier 3**: ~50,000-80,000 tokens (5-8 full transcripts)

**In `--deep` mode**: Budget shifts to ~60-96K tokens for 5-8 full transcripts.

## Step 1.5: Interactive Refinement

After initial load, present:

```
## Loaded Context Summary

**Tier 1 (Discovery)**: [N] meetings found
**Tier 2 (Summaries)**: [M] meetings loaded (~[X] tokens)
**Tier 3 (Full Transcripts)**: [K] meetings loaded (~[Y] tokens)

**Currently in deep context:**
1. [Meeting title] ([date]) - Full transcript
...

**Options:**
- "load more" - Add next 5 highest-ranked transcripts
- "swap 3 for 15" - Replace meeting #3 with meeting #15
- "focus on [participant]" - Prioritize meetings with specific person
- "focus on [date range]" - Prioritize meetings from specific period
- "continue" - Proceed with current context
```

## Context Management Commands

Throughout the session, support dynamic context management:

| Command | Action |
|---------|--------|
| `load [meeting #]` | Load full transcript for specific meeting |
| `unload [meeting #]` | Remove transcript to free context space |
| `swap [#] for [#]` | Replace one transcript with another |
| `expand [topic]` | Search for more meetings on sub-topic |
| `focus [person/date]` | Re-prioritize by participant or date range |
| `context status` | Show current token usage and loaded meetings |

## Step 1.6: Process Video Context (if --video provided)

If a video URL is provided:

1. **Try browser automation** (Chrome DevTools MCP or browser-use MCP if available)
   - Navigate to the video URL
   - Auto-screenshot at 30-second intervals
   - Extract transcript/captions if available
   - Present screenshots: "Which moments are relevant?"
   - User confirms which to include

2. **If video platform provides transcript API** (e.g., Loom, YouTube):
   - Extract auto-generated transcript directly
   - No browser automation needed

3. **Fallback**: If video is inaccessible, ask user to describe key moments or paste transcript excerpts.

**Token budget for video**: ~5,000-15,000 tokens depending on length.
