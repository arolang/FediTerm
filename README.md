# FediTerm

A Mastodon client for the terminal — written entirely in [ARO](https://aro-lang.dev), with no plugins or external dependencies required.

Browse your home timeline, watch new posts arrive in real time, compose and send toots, and reply to posts — all without leaving the terminal.

---

## Features

- **Live timeline** — streams your home feed via Mastodon's Server-Sent Events endpoint; new posts appear at the top automatically
- **Scrollable post list** — navigate with arrow keys, with a visible cursor and post counter
- **Compose** — press `n` to write a new post with a live 500-character counter
- **Reply** — press `r` on any post to reply inline
- **First-run setup** — if no config file exists, FediTerm walks you through entering your instance URL and access token interactively
- **Zero dependencies** — uses only native ARO actions: `Request`, `Stream`, `Write`, `Read`, `Listen`

---

## Screenshot

```
 FediTerm  ↑↓ navigate · n compose · r reply · esc quit
──────────────────────────────────────────────────────────
▶ @alice@mastodon.social  Alice Wonderland  2025-03-17
  Just pushed a new release — feedback welcome!
  ──────────────────────────────────────────────
  @bob@fosstodon.org      Bob Builder       2025-03-17
  Building something cool with ARO today.
  ──────────────────────────────────────────────
 42 posts · pos 0 (autoupdate)
```

---

## Requirements

- [ARO](https://aro-lang.dev) runtime (`aro` on your `PATH`)
- A Mastodon account with an access token that has `read` and `write:statuses` scopes

---

## Getting an Access Token

Log in to your Mastodon instance and go to **Preferences → Development → New Application**:

- **Application name**: FediTerm (or any name you like)
- **Scopes**: `read` + `write:statuses`
- Click **Submit** and copy the **Your access token** value

Or via the API:

```bash
# 1. Register the app
curl -X POST https://mastodon.social/api/v1/apps \
  -F 'client_name=FediTerm' \
  -F 'redirect_uris=urn:ietf:wg:oauth:2.0:oob' \
  -F 'scopes=read write:statuses'
# Note the client_id and client_secret from the response

# 2. Get a token
curl -X POST https://mastodon.social/oauth/token \
  -F 'grant_type=client_credentials' \
  -F 'client_id=<client_id>' \
  -F 'client_secret=<client_secret>' \
  -F 'scope=read write:statuses'
# Copy the access_token from the response
```

---

## Running

```bash
aro run /path/to/FediTerm
```

**First run:** FediTerm enters setup mode and prompts for your instance URL and access token. After confirming, it saves `~/.fediterm.json` and loads your timeline.

**Subsequent runs:** Config is read from `~/.fediterm.json` and the timeline loads immediately.

To reset and re-run setup:

```bash
rm ~/.fediterm.json
```

---

## Keyboard Controls

| Key         | Action                                    |
|-------------|-------------------------------------------|
| `↑`         | Scroll up / move cursor up                |
| `↓`         | Scroll down / move cursor down            |
| `n`         | Open compose dialog                       |
| `r`         | Reply to the selected post                |
| `Esc`       | Quit (timeline) · Cancel (compose/reply)  |
| `Enter`     | Send post (compose/reply) · Confirm (setup) |
| `Backspace` | Delete last character                     |

---

## Config File

`~/.fediterm.json` is created automatically on first run:

```json
{
  "instance_url": "https://mastodon.social",
  "access_token": "your-token-here"
}
```

---

## Project Structure

```
FediTerm/
├── main.aro          — Entry point: reads config, starts setup or timeline
├── lifecycle.aro     — Application-End handlers (clean shutdown)
├── timeline.aro      — Timeline fetch, SSE stream handler, post storage
├── observer.aro      — ui-state-repository Observer: renders all screen modes
├── handlers.aro      — KeyPress handler: timeline, compose, reply, and setup
├── openapi.yaml      — Schema definitions (UIState, Post, TerminalStyles, …)
└── templates/
    ├── timeline.screen — Main post list
    ├── compose.screen  — New post dialog
    ├── reply.screen    — Inline reply dialog
    └── setup.screen    — First-run token entry
```

---

## Architecture

FediTerm is built on ARO's event-driven model. The application state lives in repositories; UI re-renders automatically whenever state changes.

```
~/.fediterm.json ──exists?──► [Application-Start]
                                      │
                         ┌────────────┴────────────┐
                       yes                         no
                         │                          │
                 Store config-repository      UIState mode="setup"
                 UIState mode="timeline"      Keyboard collects
                 Emit <StartTimeline>         instance URL, then token
                 Open SSE stream             → Write ~/.fediterm.json
                                             → Switch to timeline
                         │
              [StartTimeline Handler]
                 GET /api/v1/timelines/home?limit=40
                 Emit <PostReceived> for each status
                 → Store posts (idx 0..N-1)
                 → Update UIState (latestId, totalPosts)

              [mastodon-event Handler]  (SSE — fires on each new post)
                 Prepend new post at idx 0
                 Shift existing posts by +1
                 Maintain scroll position

              [ui-state-repository Observer]
                 mode "timeline" → timeline.screen
                 mode "compose"  → compose.screen  (500-char counter)
                 mode "reply"    → reply.screen     (inline, below cursor)
                 mode "setup"    → setup.screen

              [KeyPress Handler]
                 timeline → ↑↓ scroll · n compose · r reply · esc quit
                 compose  → type · backspace · esc cancel · enter POST
                 reply    → type · backspace · esc cancel · enter POST
                 setup    → type · backspace · esc quit · enter confirm
```

---

## Troubleshooting

**`401 Unauthorized`** — Your token is invalid or expired. Regenerate it in Mastodon preferences.

**`403 Forbidden`** — Your token is missing required scopes. Re-create the application with `read write:statuses`.

**No posts appear** — Verify your token manually:

```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
  https://mastodon.social/api/v1/timelines/home?limit=1
```

**Post sent but not visible** — Your instance may require the `write:statuses` scope to be explicitly granted.
