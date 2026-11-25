# Changelog

## [1.2.0] - 2026-11-25

Improvements carried over from day-to-day use of the skill, rewritten to use config settings.

### Added
- **Meeting source detection (SKILL.md Phase 0).** Tries the provider's cloud MCP, then a local MCP, then an exported archive folder (`providers.meetings.config.archive_dir`), then a notes folder, and tells you which one it used.
- **Archive-direct loading** in `reference/tiered-loading.md` for runtimes with no meeting MCP.
- **Source health from data.** The meeting source is only reported as stale when its last sync is over 24h old, measured each run, never assumed from memory.
- **Smarter cache checks.** Extraction caches can carry `freshness` metadata from automated runs. The skill now also offers to build on an existing artifact in `paths.review_queue` and to skip re-synthesis when a fresh synthesis cache exists.
- **Wider discovery.** Searches the topic plus 2-3 variations and removes duplicates by meeting ID.
- **Scan mode: full reads of design-relevant meetings.** Keyword scans only decide relevance. Relevant meetings are read in full, and "already has a ticket" no longer counts as "nothing new".
- **Scan mode: task-tracker freshness rules.** Under 2h reuse, 2-24h delta query, over 24h or missing full query (scheduled checks can fail silently).
- **Scan mode: layered chat scan.** Connected search, then a browser pass for DMs, threads and @mentions (`providers.communication.config.people`), then a merge.
- **Agent Team model defaults per role** (`agents.models`) and an optional **design-principles check** before prototyping (`design.principles_skill`).

### Changed
- `README.md` is now the public README (previously `README-external.md`). The internal README was removed from the public repo.
- Install and download links point at `Lorlah/design-action-plugin` (the repo moved from `lolasalehu-rgb`; old links redirect).

### Fixed
- Phase 3 pointed to `reference/team-execution.md`, which doesn't exist. It now points to the Team Execution section of `reference/production-extraction.md`.
- README install "Option 2" and one download URL pointed at a private repo and returned 404 for public users.

## [1.1.0] - 2026-06-02

### Fixed
- **Granola heartbeat never fired on current Granola versions.** The watch path was hardcoded to `cache-v4.json`; Granola has since moved to v6/v7 and, as of **v7.277 (May 2026), encrypts the cache** (`cache-v*.json.enc`, leaving the plaintext file an empty stub). The heartbeat now auto-detects the newest cache artifact (encrypted or not) in both `heartbeat.sh` and `install-automation.sh`.
- **Heartbeat count-gate could skip forever on an unparseable watch source.** Added graceful degradation: when the watched file can't be counted (encrypted cache, or any unparseable format), the count gate is treated as *inconclusive* and falls through to the timestamp/throttle gate instead of silently exiting. Benefits any provider, not just Granola. The stored meeting count is no longer overwritten with a spurious `0`.

### Added
- **Official Granola Cloud MCP** (`granola-cloud`, `https://mcp.granola.ai/mcp`) documented in `docs/providers.md` as the recommended Granola server — it talks to Granola's API and is unaffected by local-cache encryption. The community local-cache MCP is retained as a legacy option for pre-v7.277 builds.
- Troubleshooting guidance: distinguish a *quiet window* (zero new meetings is normal) from a *broken source* using a data-first freshness check, rather than assuming an expired subscription. Provider-general, with the Granola encryption change documented as one specific cause.

### Notes
- These changes are scoped to the **Granola** provider path and provider-general robustness; Otter, Fireflies, Google Meet, Notion, and manual providers are unaffected by the Granola cache change (they read cloud APIs or local files).

## [1.0.0] - 2026-03-08

### Added
- Core skill: `/design-action` with 5-phase methodology (gather evidence → synthesize → suggest → create → automate)
- Provider-agnostic architecture with dispatch tables for meetings, videos, tasks, communication, and design tools
- Meeting providers: Granola, Otter.ai, Fireflies.ai, Google Meet, Notion, manual markdown
- Task providers: Jira, Linear, GitHub Issues, Notion
- Communication providers: Slack (MCP + browser), Discord, Microsoft Teams
- Design tool providers: Figma, Penpot
- Interactive setup wizard (`/setup`) with MCP auto-detection
- Scan command (`/scan`) for proactive source discovery
- Briefing command (`/briefing`) for daily design briefings
- Configurable scoring framework with weighted dimensions
- Design Decision Record (DDR) templates
- Tiered loading system for efficient token management
- Cross-meeting synthesis patterns
- Evidence grounding standards with citation formats
- 9 artifact templates (journey maps, prototypes, wireframes, research synthesis, demo videos, FigJam boards, prioritization matrices, flow diagrams)
- Production pattern extraction for prototype fidelity
- Agent Team dispatch for parallel execution
- Config validation script
- Comprehensive provider setup documentation
- Automation scripts: daily-briefing.sh, heartbeat.sh, task-check.sh (all config-driven)
- Cross-platform installer: install-automation.sh (macOS launchd, Linux systemd, cron fallback)
- Scheduler templates: 3 launchd plists, 6 systemd units, 1 crontab template
- Heartbeat with 3-gate system (time, count, timestamp) before invoking Claude
- Decision-log auto-generation from DDR files via embedded Python
