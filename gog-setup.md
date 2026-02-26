# gog CLI — Google Workspace Integration

`gog` is a CLI for Gmail, Calendar, Drive, Contacts, Docs, and Sheets.

## Install

```bash
brew install gog   # or via openclaw onboard
```

## Auth Setup (Remote/Headless)

`gog` has a built-in remote auth flow that handles the OAuth dance.
See [google-oauth.md](./google-oauth.md) for background.

### Step 1: Register credentials
```bash
gog auth credentials /path/to/client_secret.json
```

### Step 2: Start auth (remote mode)
```bash
GOG_KEYRING_PASSWORD=YOUR_PASSWORD \
gog auth add YOUR@gmail.com \
  --services gmail,calendar,drive,contacts,docs,sheets \
  --remote --step 1
```

This prints an auth URL. User visits it, clicks Allow, gets "Site can't be reached".
They copy the full `http://127.0.0.1:PORT/oauth2/callback?...` URL.

⚠️ **Note the port** — it changes each run. Use only the URL printed by step 1.

### Step 3: Complete auth
```bash
GOG_KEYRING_PASSWORD=YOUR_PASSWORD \
gog auth add YOUR@gmail.com \
  --services gmail,calendar,drive,contacts,docs,sheets \
  --remote --step 2 \
  --auth-url "http://127.0.0.1:PORT/oauth2/callback?state=...&code=..."
```

### Persist keyring password
```bash
echo 'export GOG_KEYRING_PASSWORD=YOUR_PASSWORD' >> ~/.bashrc
```

## Usage

Prefix every command with `GOG_KEYRING_PASSWORD=YOUR_PASSWORD` or set the env var.

```bash
# Gmail — search unread non-starred
gog gmail search 'is:unread -is:starred' --max 20 --account you@gmail.com

# Gmail — mark thread as read
gog gmail thread modify THREAD_ID --remove UNREAD --account you@gmail.com --force

# Calendar — tomorrow's events (all calendars)
gog calendar events --tomorrow --all --account you@gmail.com

# Calendar — create event
gog calendar create primary --summary "Meeting" --from "2026-03-01T10:00" --to "2026-03-01T11:00"

# Drive — list files
gog drive list --account you@gmail.com

# Contacts — search
gog contacts search "John" --account you@gmail.com
```

## Supported Services

`gmail` | `calendar` | `drive` | `contacts` | `docs` | `sheets` | `chat` |
`classroom` | `slides` | `tasks` | `people` | `forms` | `appscript` | `groups` | `keep`

Note: **YouTube is NOT supported** by gog. Use the YouTube Data API directly
with a custom OAuth token. See [youtube-oauth.md](./youtube-oauth.md).

## Gotchas

- **`api.runpod.ai` not `api.runpod.io`** — unrelated but came up in same session
- **State mismatch**: if step 1 and step 2 use different states (e.g. user visited old URL), re-run step 1 to generate a fresh state
- **Keyring password**: must match on every command; set in env for convenience
- **API not enabled**: 403 `accessNotConfigured` = go to Google Cloud Console and enable the specific API
- **Re-auth for new scopes**: tokens are not additive — to add a new service you must re-auth with all desired services listed
