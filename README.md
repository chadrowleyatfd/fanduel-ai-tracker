# Claude Pipeline — AIG Jira Dashboard

A lightweight static dashboard tracking Claude-related and MCP-connector intake tickets in FanDuel's AIG Jira project. Built to be hosted on GitHub Pages and shared with the team.

## What it shows

The dashboard pulls a snapshot of tickets matching this Jira filter:

```
project = "AIG" AND (text ~ "Claude" OR text ~ "connector" OR text ~ "MCP") ORDER BY updated DESC
```

It surfaces:
- Five KPI cards (Total, Claude-specific, Awaiting Review, Approved, Canceled).
- Six filter pills (default: **Claude-specific**). "Claude-specific" matches tickets whose summary references Claude, Anthropic, Opus 4.x, Sonnet 4.x, Haiku 4.x, or Cowork — so model-approval tickets like `AIG-177` (Opus 4.6 / Sonnet 4.6) are included even though the word "Claude" isn't in the title.
- A searchable, sortable, paginated table of every ticket, with direct links into Jira.

## How to host on GitHub Pages

1. Create a new repo on GitHub (private is fine — Pages supports private repos for Enterprise accounts).
2. Drop `index.html` and `README.md` in the repo root and push to `main`.
3. In repo **Settings → Pages**, set source to `main` / root, save.
4. GitHub gives you a URL like `https://<org>.github.io/<repo>/`. Share that with the team.

The dashboard is fully self-contained — Grid.js loads from a public CDN, everything else (data + styles + logic) is inlined in `index.html`. No build step.

## Refreshing the data (manual, for now)

The current data is baked into `index.html` as a JSON literal inside a `<script>` tag (search for `const ISSUES =`). To refresh:

1. Ping Claude in Cowork and ask for a new snapshot ("regenerate the AIG dashboard data").
2. Claude will hit Jira through the Atlassian MCP, regenerate `index.html`, and drop it in your outputs folder.
3. Replace `index.html` in this repo, commit, push. GitHub Pages picks up the change in a minute or two.

## Future: weekly auto-refresh via GitHub Actions

When you're ready to automate (and have a Jira API token you can store as a repo secret), the rough plan is:

1. Add a GitHub Actions workflow on a Monday-AM cron (`0 13 * * 1` = 8 AM ET).
2. The workflow runs a small Python script that calls the Jira REST API (`/rest/api/3/search/jql`) with the same JQL filter.
3. The script regenerates `index.html` (or a separate `data.js`) and commits the change back to the repo.
4. GitHub Pages auto-publishes. Team always sees fresh data on Monday morning.

Secrets needed:
- `JIRA_EMAIL` — your Atlassian account email.
- `JIRA_API_TOKEN` — generated at https://id.atlassian.com/manage-profile/security/api-tokens.
- `JIRA_CLOUD_ID` — `fanduel.atlassian.net`.

Drop a line in Cowork when you want this scaffolded — it's about 60 lines of Python plus a 20-line workflow YAML.

## Access

The dashboard is gated by a client-side password. Share the URL + password with the team.

- **Password:** `fanduelaiupdates`

Once entered, the password is remembered for the rest of the browser session (via `sessionStorage`), so users won't be re-prompted on refresh. Closing the tab and reopening will require it again.

### A note on what this password actually protects

The password check is client-side: the password's SHA-256 hash is embedded in `index.html`, and the gate is enforced in JavaScript. **Anyone who views source can find the hash and either reverse-engineer it or skip the gate entirely.** Treat this as obscurity (a soft "don't share with people outside the team"), not real security.

For the data on this dashboard — Jira ticket metadata that's already visible to anyone with AIG project access — that level of protection is fine. If you ever put truly sensitive data on a GH Pages dashboard, front it with Cloudflare Access, an internal SSO proxy, or move it off GH Pages entirely.

### Changing the password

1. Generate a SHA-256 hash of your new password: `echo -n "newpassword" | shasum -a 256` (Mac) or any online SHA-256 tool.
2. Replace the `PASSWORD_HASH` constant near the bottom of `index.html` with the new hash.
3. The password check normalizes input to lowercase and trims whitespace before hashing, so generate the hash from the lowercased version (`echo -n "newpassword" | shasum -a 256`, all lowercase).
4. Commit and push.

## Discovery hardening

A few small mitigations are baked in to reduce the chance that the URL gets surfaced where it shouldn't:

- `<meta name="robots" content="noindex, nofollow, noarchive, nosnippet">` and an explicit `googlebot` directive tell well-behaved search engines not to index, follow, archive, or snippet the page.
- `robots.txt` at the repo root disallows all crawlers as a belt-and-suspenders layer.
- `<meta name="referrer" content="no-referrer">` prevents outbound clicks (e.g., to Jira tickets) from leaking the dashboard URL in the `Referer` header.

These don't protect the data — they reduce the chance the URL gets discovered in the first place. Combined with the password gate, that's the realistic security envelope for a static GH Pages dashboard.

## File map

```
.
├── index.html   # the dashboard (data baked in)
├── robots.txt   # disallows crawlers
└── README.md    # this file
```
