# Nomio Status

Status messages the Nomio iOS app fetches at runtime, so an outage or a maintenance window can be
announced without shipping a build.

Two files, one per build configuration:

| File | Who sees it |
|---|---|
| `status-debug.json` | Debug builds (developers). Safe to experiment in. |
| `status-production.json` | Release builds — TestFlight and the App Store. Real users. |

The app reads them straight from `raw.githubusercontent.com` on this repo's `main` branch. Merging
to `main` publishes; there is no other step.

This repo is public and separate from the app repo on purpose. `pwdvandy/Nomio` is private and
`raw.githubusercontent.com` will not serve from it without a token shipped in the binary. It is also
deliberately off Supabase: a pill announcing that Supabase is down is worth nothing if fetching it
needs Supabase.

## What a pill looks like

Each entry is a small capsule floating above the tab bar. Tapping it expands into a card listing the
detail rows. Collapsed, only the title shows — so the title is a label, not a sentence.

## Format

```json
{
  "pills": [
    {
      "id": "supabase-outage",
      "severity": "error",
      "title": "Service issues",
      "rows": [
        {
          "icon": "danger",
          "title": "Recipes may not load",
          "caption": "We're having trouble reaching our servers. We're on it."
        },
        {
          "icon": "cart",
          "title": "Groceries still work",
          "caption": "Your lists are on this device and sync once we're back."
        }
      ]
    }
  ]
}
```

Nothing to announce is an empty array, not an empty file:

```json
{ "pills": [] }
```

### Fields

| Field | Required | Notes |
|---|---|---|
| `pills` | yes | Top-level array. Missing key = the whole file is unreadable, see below. |
| `pills[].id` | yes | Stable string identifying the *condition*, not the post. Keep it the same while the condition lasts; the app uses it to tell "still down" from "down again" and to keep an expanded card open when another pill arrives. Do not use `offline` (see Reserved ids). |
| `pills[].severity` | yes | `info`, `warning` or `error`. Anything else drops that pill. |
| `pills[].title` | yes | The collapsed capsule's text. One to three words. "Service issues", "Maintenance", "New recipes weekly". |
| `pills[].rows` | no | Detail rows in the expanded card. Omit for a badge with nothing more to say. |
| `rows[].icon` | yes | One name from the allowlist below. |
| `rows[].title` | yes | Row heading, a short line: "Groceries still work". |
| `rows[].caption` | yes | One or two sentences. Wraps freely, no length limit — but the card grows with it. |

### Severity

| Value | Colour | Use for |
|---|---|---|
| `error` | red | Something is broken right now and the user will hit it. |
| `warning` | orange | Degraded, or a mode change rather than a failure. |
| `info` | blue | Announcements, upcoming maintenance, nothing broken. |

Severity also sets order: the most severe pill sits nearest the tab bar, read first. **At most three
pills show at once**, and the least severe are the ones dropped, so an `info` pill added during an
outage may never be seen.

### Icons

Only these names render. Anything else silently becomes `info_circle`, so typos degrade rather than
leaving a blank circle — but they still lose you the right icon, so check the spelling.

`danger` · `info_circle` · `shield` · `timer` · `cart` · `search` · `book` · `bell` · `rocket` ·
`refresh_left`

### Reserved ids

`offline` belongs to the app's own built-in pill, shown whenever the device has no connection. A
feed pill using that id is ignored. Everything else is yours.

## How the app behaves

**When it fetches** — at launch, on every foreground, and on every reconnect. GitHub's CDN caches
for five minutes, so expect **up to ~5 minutes** between merging and users seeing a change. Plan the
maintenance-window post accordingly, and do not treat the feed as a real-time channel.

**When the fetch or the JSON fails** — the app keeps whatever pills are already on screen and says
nothing. This is the outage announcer; an error about the announcer would be noise. The practical
consequence: **a malformed file does not clear a stale pill.** If you break the JSON while an
announcement is live, users keep seeing the old one until you fix the file.

**Clearing an announcement** — publish `{ "pills": [] }`. Deleting the file or leaving it empty does
not clear anything, it just fails to parse.

**Partial validation** — a pill with an unreadable `severity` is dropped and the rest of the file
still applies. So one bad entry costs you that entry, not the whole feed.

**Where they appear** — pills are part of the signed-in tab bar, so a signed-out user on the login
screen sees none of them.

## Publishing checklist

1. Edit `status-production.json` (or `status-debug.json` to rehearse).
2. Validate before pushing — a broken file silently keeps the previous state:
   ```bash
   jq . status-production.json
   ```
3. Merge to `main`. Wait ~5 minutes for the CDN.
4. When the incident ends, set `pills` back to `[]`. An outage pill nobody cleared is worse than no
   pill at all.

## Writing the copy

Users read this while something is broken, so lead with what still works. The built-in offline pill
is the house style:

> **Groceries keep working** — Edits are saved on this device and sync once you're back online.
> **Cooking keeps going** — Your active meal and its step timers run entirely on this device.
> **Exploring has to wait** — New meals, search and images need a connection to load.

One row per capability the user might be wondering about. Name the thing, say whether it works, say
what happens to their data. No apologising at length, no internal cause ("Postgres connection pool
exhausted"), no ETA you are not sure of.
