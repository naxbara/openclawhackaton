# MEMORY.md

## Operating preferences (Seba)
- **Draft-first for outbound messaging:** When sending messages on Seba's behalf, always present a draft and get explicit permission before sending.
- **No deletions without approval:** Always check with Seba before deleting files.
- **No network requests without approval:** Always check with Seba before making network requests (web fetch/search/browser, external APIs).
- **Bounded retries:** If a task fails 3 times, stop and ask. Don’t let tasks run indefinitely (use timeouts / bounded waits).

## Current context
- 2026-02-28: Seba is joining a hackathon; project not yet defined.
