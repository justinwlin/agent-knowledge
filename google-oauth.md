# Google OAuth — Manual Flow (Headless/Remote Environments)

When running on a remote server (no browser access), the standard OAuth redirect
to `localhost` fails because the callback never reaches the server.

## Setup

### 1. Create a Google Cloud project + OAuth credentials

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a project (or use existing)
3. Enable the APIs you need (Gmail, Calendar, YouTube Data API v3, etc.)
4. **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
5. Application type: **Desktop app**
6. Download the JSON → `client_secret.json`

The JSON looks like:
```json
{
  "installed": {
    "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
    "client_secret": "GOCSPX-...",
    "redirect_uris": ["http://localhost"]
  }
}
```

### 2. Enable required APIs

Each Google service needs its API enabled separately:
- Gmail: `gmail-json.googleapis.com`
- Calendar: `calendar-json.googleapis.com`
- YouTube: `youtube.googleapis.com`
- Drive: `drive.googleapis.com`

Enable at: `https://console.developers.google.com/apis/api/<API_ID>/overview?project=<PROJECT_ID>`

## The Remote OAuth Dance

Since the redirect goes to `http://localhost` (which is on the user's machine,
not the server), we use a 2-step manual flow.

### Step 1: Generate the auth URL

```bash
CLIENT_ID="YOUR_CLIENT_ID.apps.googleusercontent.com"
SCOPE="https://www.googleapis.com/auth/gmail.modify"  # adjust per service
STATE=$(openssl rand -base64 24 | tr -d '=/+' | head -c 32)

URL="https://accounts.google.com/o/oauth2/v2/auth?\
client_id=${CLIENT_ID}&\
redirect_uri=http://localhost&\
response_type=code&\
scope=${SCOPE}&\
access_type=offline&\
prompt=consent&\
state=${STATE}"

echo $URL
```

### Step 2: User visits the URL, gets "Site can't be reached"

After clicking Allow, the browser tries to load:
```
http://localhost/?code=4/0AfrIep...&state=...
```

This fails (nothing listening on localhost) but the **URL in the address bar
contains the auth code**. User copies and pastes the full URL back.

### Step 3: Extract code + exchange for tokens

```bash
# Parse the code from the callback URL
CODE="4/0AfrIep..."  # extracted from pasted URL

curl -s -X POST "https://oauth2.googleapis.com/token" \
  -d "code=$CODE" \
  -d "client_id=YOUR_CLIENT_ID.apps.googleusercontent.com" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "redirect_uri=http://localhost" \
  -d "grant_type=authorization_code"
```

Returns:
```json
{
  "access_token": "ya29...",
  "expires_in": 3599,
  "refresh_token": "1//03...",
  "scope": "...",
  "token_type": "Bearer"
}
```

**Save both `access_token` and `refresh_token`.** The refresh token is long-lived
and lets you get new access tokens without user interaction.

## Refreshing Tokens

Access tokens expire after ~1 hour. Use the refresh token to get a new one:

```bash
curl -s -X POST "https://oauth2.googleapis.com/token" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "refresh_token=YOUR_REFRESH_TOKEN" \
  -d "grant_type=refresh_token"
```

## Multiple Scopes

Combine scopes with spaces (URL-encoded as `+`):
```
scope=https://www.googleapis.com/auth/gmail.modify+https://www.googleapis.com/auth/calendar
```

## Gotchas

- **`prompt=consent`** — required to get a refresh token on subsequent auths; without it Google may not return one
- **`access_type=offline`** — required for refresh tokens
- **API not enabled** — 403 `accessNotConfigured` means you need to enable the API in Cloud Console
- **Scope not in token** — if you add a new scope later, you must re-auth with all scopes combined; tokens are not additive
- **State mismatch** — if using a tool like `gog` that stores state internally, you must use the URL it generated; visiting an old URL causes state mismatch errors
