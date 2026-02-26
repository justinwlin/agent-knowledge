# YouTube Data API v3 — OAuth + Usage

## Auth Setup

YouTube uses standard Google OAuth. See [google-oauth.md](./google-oauth.md) for the full dance.

**Scope needed:** `https://www.googleapis.com/auth/youtube.readonly`

**API to enable:** `youtube.googleapis.com` in Google Cloud Console

## Token Storage

Store tokens at `~/.config/youtube/tokens.json`:
```json
{
  "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
  "client_secret": "YOUR_CLIENT_SECRET",
  "refresh_token": "1//03...",
  "access_token": "ya29..."
}
```

## Token Refresh Helper Script

Save at `~/.config/youtube/get_token.sh`:

```bash
#!/bin/bash
TOKENS=$(cat ~/.config/youtube/tokens.json)
CLIENT_ID=$(echo $TOKENS | jq -r '.client_id')
CLIENT_SECRET=$(echo $TOKENS | jq -r '.client_secret')
REFRESH_TOKEN=$(echo $TOKENS | jq -r '.refresh_token')

RESP=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "refresh_token=$REFRESH_TOKEN" \
  -d "grant_type=refresh_token")

ACCESS_TOKEN=$(echo $RESP | jq -r '.access_token')
jq --arg t "$ACCESS_TOKEN" '.access_token = $t' ~/.config/youtube/tokens.json > /tmp/yt_tmp.json
mv /tmp/yt_tmp.json ~/.config/youtube/tokens.json
echo $ACCESS_TOKEN
```

Usage: `TOKEN=$(~/.config/youtube/get_token.sh)`

## Common API Calls

### List all playlists (including private)
```bash
TOKEN=$(~/.config/youtube/get_token.sh)
curl -s "https://www.googleapis.com/youtube/v3/playlists?part=snippet,contentDetails&mine=true&maxResults=50" \
  -H "Authorization: Bearer $TOKEN" | jq '[.items[] | {id: .id, title: .snippet.title}]'
```

### List videos in a playlist
```bash
TOKEN=$(~/.config/youtube/get_token.sh)
PLAYLIST_ID="PLxxx..."
curl -s "https://www.googleapis.com/youtube/v3/playlistItems?part=snippet,contentDetails&playlistId=$PLAYLIST_ID&maxResults=50" \
  -H "Authorization: Bearer $TOKEN" | jq '[.items[] | {
    videoId: .contentDetails.videoId,
    title: .snippet.title,
    publishedAt: .snippet.publishedAt,
    url: ("https://www.youtube.com/watch?v=" + .contentDetails.videoId)
  }]'
```

### Paginate through large playlists
```bash
# First page returns nextPageToken; pass it as &pageToken=TOKEN for subsequent pages
curl -s "https://www.googleapis.com/youtube/v3/playlistItems?part=snippet,contentDetails&playlistId=$PLAYLIST_ID&maxResults=50&pageToken=PAGE_TOKEN" \
  -H "Authorization: Bearer $TOKEN"
```

## Polling for New Videos (Change Detection)

Store a state file of known video IDs. On each poll, compare current playlist
to stored state and process only new items.

```bash
STATE_FILE="~/.config/youtube/playlist_state_${PLAYLIST_ID}.json"

# Load known IDs
KNOWN=$(cat $STATE_FILE 2>/dev/null || echo '[]')

# Fetch current playlist
CURRENT=$(curl -s "https://www.googleapis.com/youtube/v3/playlistItems?..." \
  -H "Authorization: Bearer $TOKEN" | jq '[.items[].contentDetails.videoId]')

# Find new IDs (in current but not in known)
NEW=$(jq -n --argjson known "$KNOWN" --argjson current "$CURRENT" \
  '$current - $known')

# Process new videos...

# Update state
echo $CURRENT > $STATE_FILE
```

## Gotchas

- `contentDetails.itemCount` is null unless you request `contentDetails` in the `part` parameter
- Private playlists require OAuth; public ones work with just an API key
- Playlist items API returns max 50 per page; use `nextPageToken` to paginate
- The YouTube API quota resets daily; list operations cost 1 unit each (very cheap)
- `youtube.readonly` scope covers: playlists, playlist items, video metadata, channel info
