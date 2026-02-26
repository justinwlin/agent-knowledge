# RunPod — Serverless Transcription + Summarization

Endpoint built on WhisperX + DeepSeek. Accepts YouTube URLs or audio file URLs,
returns transcription with timestamps + AI summary.

Repo: https://github.com/justinwlin/transcribe-and-summarize-on-runpod

## Endpoint

- **ID:** `dukb8676whyijz`
- **Submit:** `POST https://api.runpod.ai/v2/dukb8676whyijz/run`
- **Status:** `GET https://api.runpod.ai/v2/dukb8676whyijz/status/{job_id}`

⚠️ Domain is `api.runpod.ai` — NOT `api.runpod.io` (that returns 404)

## Submit a Job

```bash
RUNPOD_KEY="YOUR_API_KEY"

curl -s -X POST "https://api.runpod.ai/v2/dukb8676whyijz/run" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $RUNPOD_KEY" \
  -d '{
    "input": {
      "audio_file": "https://www.youtube.com/watch?v=VIDEO_ID",
      "language": "en"
    }
  }'
# Returns: { "id": "JOB_ID", "status": "IN_QUEUE" }
```

Accepts: YouTube URLs, YouTube playlist URLs, direct audio file URLs (.mp3, .wav, .m4a, etc.)

## Poll for Completion

```bash
JOB_ID="abc123"
curl -s "https://api.runpod.ai/v2/dukb8676whyijz/status/$JOB_ID" \
  -H "Authorization: Bearer $RUNPOD_KEY"
```

Status flow: `IN_QUEUE` → `IN_PROGRESS` → `COMPLETED` or `FAILED`

Typical execution time: ~24 seconds on GPU

## Polling Loop

```bash
for i in $(seq 1 40); do
  RESP=$(curl -s "https://api.runpod.ai/v2/dukb8676whyijz/status/$JOB_ID" \
    -H "Authorization: Bearer $RUNPOD_KEY")
  STATUS=$(echo $RESP | jq -r '.status')
  echo "[$i] $STATUS"
  if [[ "$STATUS" == "COMPLETED" || "$STATUS" == "FAILED" ]]; then
    echo $RESP | jq '.'
    break
  fi
  sleep 15
done
```

## Response (COMPLETED)

```json
{
  "status": "COMPLETED",
  "output": {
    "file_path": "Video Title.mp3",
    "detected_language": "en",
    "full_text": "Full transcript here...",
    "segments": [
      { "start": 0.0, "end": 5.2, "text": "First segment..." },
      ...
    ],
    "summarized": "AI-generated summary with structured bullet points..."
  }
}
```

## Input Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `audio_file` | string | required | YouTube URL or audio file URL |
| `language` | string | null | Language code (e.g. "en"). Auto-detected if omitted |
| `align_output` | bool | false | Return word-level timestamps |
| `batch_size` | int | 64 | Processing batch size |
| `debug` | bool | false | Enable debug mode |

## Known Failure Modes

### CUDA Out of Memory (OOM)
```
Error processing file /tmp/.../file.mp3: CUDA failed with error out of memory
```
**Cause:** Worker GPU ran out of VRAM for the Whisper model.
**Fix:** Skip the video and move on. All retries go to the same worker and will also fail.
**Long-term fix:** Upgrade endpoint GPU type (need ≥16GB VRAM: RTX 3090, A4000, or better)
or reduce Whisper model size in the handler.

### FAILED (general)
Skip the row/video, log it, continue to the next.

## Concurrent Requests (Batching)

RunPod is serverless/on-demand — it queues and scales automatically. You can
submit multiple jobs simultaneously without waiting for previous ones to finish.

Recommended pattern: submit up to **10 jobs at once**, collect all job IDs,
then poll all of them in a loop until each is COMPLETED or FAILED.

```bash
# Submit 10 jobs, collect IDs
JOB_IDS=()
for URL in "${URLS[@]}"; do
  JOB_ID=$(curl -s -X POST "https://api.runpod.ai/v2/dukb8676whyijz/run" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $RUNPOD_KEY" \
    -d "{\"input\": {\"audio_file\": \"$URL\", \"language\": \"en\"}}" | jq -r '.id')
  JOB_IDS+=("$JOB_ID")
done

# Poll all until done
for JOB_ID in "${JOB_IDS[@]}"; do
  # poll loop per job...
done
```

## No Local Storage Needed

RunPod downloads YouTube audio internally — nothing is stored on the calling server.
This makes it safe to process many videos without worrying about disk space.
