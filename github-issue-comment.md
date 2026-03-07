**Update: revised approach after further testing**

The cursor restart approach I originally proposed doesn't work. Further testing showed:

- The cursor expires at exactly 11,200 pages every time, regardless of pageSize or request rate. This is a position-based limit, not a time-based TTL.
- Restarting the search from the beginning hits the same expiry at the same position every attempt.

**Test results (same workspace, same token):**

| pageSize | Pages scanned | API calls | Time |
|----------|--------------|-----------|------|
| 25 | 11,200 | 449 | 793s |
| 100 | 11,200 | 112 | 353s |

Same expiry point regardless of pageSize. The cursor limit is positional, not time-based.

**Revised approach (two files changed):**

The root cause is that `fetchRootPages()` must scan every page in the workspace via the search API just to find the handful of workspace-level items. There's no server-side filter for `parent.type`, so for any workspace over ~11k pages, this scan will always hit the cursor limit.

The fix adds a `NOTION_ROOT_PAGE_IDS` environment variable. When set to a comma-separated list of Notion page/database IDs, `fetchRootPages()` fetches them directly via `pages.retrieve`/`databases.retrieve` instead of scanning via search. This bypasses the search API entirely -- the tree traversal in `parseChildPages()` already handles discovering all child content recursively, so only the top-level entry points are needed.

When the env var is not set, the existing search behavior is preserved with two improvements:
- `pageSize` increased from 25 to 100 (Notion API maximum), reducing API calls by 4x for workspaces that fit within the cursor limit.
- On cursor expiry, the error message now directs the admin to set `NOTION_ROOT_PAGE_IDS` and explains how to find page IDs from Notion share URLs, instead of showing the raw Notion API error.

Changes are in `plugins/notion/server/notion.ts` and `plugins/notion/server/env.ts`. No interface changes to `fetchRootPages()` -- it returns the same `Page[]` either way.
