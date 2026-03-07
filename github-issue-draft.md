# Title

Notion import fails for large workspaces due to search cursor expiration

---

# Current Behavior

Notion imports fail for workspaces with ~10,000+ pages. The import fails during the initial page discovery phase with:

```
Failed: The start_cursor provided is invalid: 4d49987c-a64c-4ab8-916e-d9aeaffa265b
```

`fetchRootPages()` in `plugins/notion/server/notion.ts` paginates through all workspace pages via the Notion search API. The search cursor has an undocumented positional limit (~11,200 pages in our testing) that causes it to expire before pagination completes on large workspaces. The limit is position-based, not time-based -- the same cursor ID expires at the same page count regardless of `pageSize` or request rate.

`fetchWithRetry()` handles rate limits and timeouts but not cursor expiration, so the import fails without recovery. This is 100% reproducible on any large workspace.

---

# Expected Behavior

The Notion import should complete successfully regardless of workspace size. If a search cursor expires mid-pagination, the import should recover and continue rather than failing outright.

---

# Steps To Reproduce

1. Connect a Notion workspace containing 10,000+ pages via Settings > Integrations > Notion
2. Start an import from Settings > Import > Notion
3. Import enters "processing" state, then fails with "The start_cursor provided is invalid"
4. Every subsequent import attempt fails at the same point

---

# Environment

- Outline: 1.5.0 (Enterprise, self-hosted on ECS Fargate)
- Browser: N/A (server-side failure)
- Notion workspace: ~15,000 pages

---

# Anything else?

**Root cause:** The Notion search API has no server-side filter for `parent.type === "workspace"`, so `fetchRootPages()` must paginate through every page in the workspace to find workspace-level items. This is a Notion API limitation.

**Proposed fix (two changes to `plugins/notion/server/notion.ts`):**

1. Increase `pageSize` from 25 to 100 (Notion API maximum). Reduces API calls from ~600 to ~150 for a 15k page workspace.

2. Add cursor recovery to `fetchRootPages()`. When the cursor expires, restart the search from the beginning, skip already-collected pages via a `Set<string>` of seen IDs, and allow up to 3 restarts before failing. Log recovery events for observability.

The change is isolated to `fetchRootPages()` only. No interface changes, no other files modified. The method returns the same `Page[]` as before.

**Tested** with bare API calls against a production workspace (~15,000 pages). Successfully traversed 5,448 pages visible to the test token in 55 API calls, 88 seconds, 0 cursor restarts needed. The recovery mechanism is a safety net for workspaces large enough that even pageSize=100 is not sufficient.

I have a branch with the fix ready and can open a PR if this approach looks good.
