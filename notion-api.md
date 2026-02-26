# Notion API — Internal Integration

## Setup

1. Go to [notion.so/profile/integrations](https://notion.so/profile/integrations)
2. Click **New integration** → choose **Internal** (not Public — Public requires company info)
3. Name it (e.g. "echo"), select your workspace
4. Copy the API key (starts with `ntn_...`)

```bash
mkdir -p ~/.config/notion
echo "ntn_YOUR_KEY" > ~/.config/notion/api_key
# Also set as env var (required for the bundled OpenClaw notion skill)
export NOTION_API_KEY=ntn_YOUR_KEY  # add to ~/.bashrc + openclaw.json env.vars
```

### Share pages with the integration

Notion requires manual sharing. For each page/database:
- Open in Notion → `...` menu → **Connections** → select your integration

The integration can ONLY see what's explicitly shared with it.

## Auth Header

```bash
NOTION_KEY=$(cat ~/.config/notion/api_key)
# Header: Authorization: Bearer $NOTION_KEY
# Header: Notion-Version: 2025-09-03
```

## Common Operations

### Search all accessible pages
```bash
curl -s -X POST "https://api.notion.com/v1/search" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{"page_size": 20}' | jq '.'
```

### Query a database (with filter)
```bash
# Use data_sources endpoint for querying (faster than databases endpoint)
curl -s -X POST "https://api.notion.com/v1/data_sources/DATA_SOURCE_ID/query" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "and": [
        { "property": "URL", "url": { "contains": "youtube.com" } },
        { "property": "Quick Overview", "rich_text": { "is_empty": true } }
      ]
    },
    "page_size": 10
  }'
```

Note: `data_source_id` differs from `database_id`. Both come from the search results.

### Update a page property
```bash
curl -s -X PATCH "https://api.notion.com/v1/pages/PAGE_ID" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "Quick Overview": {
        "rich_text": [{"text": {"content": "Your summary here"}}]
      }
    }
  }'
```

### Append blocks to a page (e.g. transcript)
```bash
curl -s -X PATCH "https://api.notion.com/v1/blocks/PAGE_ID/children" \
  -H "Authorization: Bearer $NOTION_KEY" \
  -H "Notion-Version: 2025-09-03" \
  -H "Content-Type: application/json" \
  -d '{
    "children": [
      {
        "object": "block",
        "type": "heading_2",
        "heading_2": { "rich_text": [{"text": {"content": "📝 Transcript"}}] }
      },
      {
        "object": "block",
        "type": "paragraph",
        "paragraph": { "rich_text": [{"text": {"content": "Transcript chunk here..."}}] }
      }
    ]
  }'
```

## Gotchas

- **Rich text 2000 char limit** — each rich_text block is limited to 2000 chars. Split long content into multiple paragraph blocks
- **`data_sources` vs `databases`** — use `data_sources` endpoint for querying (it's what Notion uses internally); `databases` endpoint also works but may behave differently with some filter types
- **Relation properties** — when updating relation fields, pass an array of `{id: "PAGE_ID"}` objects
- **Rollup properties** — read-only, cannot be set directly
- **Internal vs Public integration** — Internal is for personal use and needs no OAuth flow; Public integration requires company website, privacy policy, etc.
