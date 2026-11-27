# Argon Audio Insights

Generate summaries, notable moments, and action-item drafts from audio files using Gemini 4 Argon through CometAPI.

> **Compatibility notice:** Confirm that your CometAPI account supports the model ID, endpoint, audio format, and upload size used below. Test the project against the current gateway before describing it as working.

## Features

- Podcast show-note drafts
- Action items, decisions, and deadlines
- Notable quote candidates
- Approximate timestamps and speaker labels
- Basic request and response error handling

Quotes, timestamps, and speaker identities can be wrong. Verify them against the original recording before publication or use.

## Setup

Requires Python 3.9 or later:

```bash
python -m pip install requests
export COMETAPI_KEY="your_api_key_here"
```

## Usage

Generate show notes:

```bash
python insights.py episode.mp3
```

Extract action items:

```bash
python insights.py call.m4a --mode actions
```

Find notable quote candidates:

```bash
python insights.py interview.wav --mode quotes
```

Available modes:

- `shownotes`
- `actions`
- `quotes`

## Script

Create `insights.py`:

```python
import argparse
import base64
import mimetypes
import os
from pathlib import Path

import requests

API_URL = (
    "https://api.cometapi.com/v1beta/models/"
    "gemini-4-argon:generateContent"
)

PROMPTS = {
    "shownotes": """
Create draft show notes using only the supplied audio.

Return:
1. A three-sentence summary
2. Five key points with approximate timestamps
3. Three short quote candidates with approximate timestamps and speakers
4. Any names or details that require verification

Do not invent inaudible words, speakers, or timestamps. Mark uncertain content
with [CHECK]. Quotes must be verified against the recording before publication.
""",
    "actions": """
Using only the supplied audio, list:
- Action items
- Decisions
- Deadlines
- The person responsible, when explicitly stated
- An approximate timestamp for each item

Do not infer an owner or deadline that was not stated. Use [CHECK] when the audio
is unclear. If no action items are present, say so.
""",
    "quotes": """
Identify up to eight notable quote candidates from the supplied audio.

For each candidate, provide:
- A short verbatim excerpt
- The speaker, if confidently identifiable
- An approximate timestamp
- [CHECK] if any wording, speaker, or timestamp is uncertain

Do not reconstruct or improve the speaker's words. Every quote must be checked
against the recording before publication.
""",
}


def parse_args():
    parser = argparse.ArgumentParser(
        description="Generate draft insights from an audio file."
    )
    parser.add_argument("file", type=Path, help="Audio file to analyze")
    parser.add_argument(
        "--mode",
        choices=PROMPTS,
        default="shownotes",
    )
    return parser.parse_args()


def extract_text(payload):
    candidates = payload.get("candidates") or []
    if not candidates:
        raise RuntimeError("The API returned no candidates.")

    parts = candidates[0].get("content", {}).get("parts", [])
    text = "\n".join(
        part["text"]
        for part in parts
        if isinstance(part, dict) and part.get("text")
    )

    if not text:
        raise RuntimeError("The API returned no text output.")

    return text


def main():
    args = parse_args()

    api_key = os.environ.get("COMETAPI_KEY")
    if not api_key:
        raise SystemExit("Set COMETAPI_KEY before running this script.")

    if not args.file.is_file():
        raise SystemExit(f"Audio file not found: {args.file}")

    mime_type, _ = mimetypes.guess_type(args.file.name)
    if not mime_type or not mime_type.startswith("audio/"):
        raise SystemExit(
            "Could not identify an audio MIME type. "
            "Use a recognized audio extension."
        )

    audio_data = base64.b64encode(args.file.read_bytes()).decode("ascii")

    body = {
        "contents": [{
            "role": "user",
            "parts": [
                {
                    "inline_data": {
                        "mime_type": mime_type,
                        "data": audio_data,
                    }
                },
                {"text": PROMPTS[args.mode].strip()},
            ],
        }],
        "generationConfig": {
            "maxOutputTokens": 3000,
            "temperature": 0.2,
        },
    }

    headers = {
        "x-goog-api-key": api_key,
        "Content-Type": "application/json",
    }

    try:
        response = requests.post(
            API_URL,
            headers=headers,
            json=body,
            timeout=900,
        )
        response.raise_for_status()
        payload = response.json()
    except requests.Timeout as exc:
        raise SystemExit("The request timed out.") from exc
    except requests.RequestException as exc:
        details = ""
        if exc.response is not None:
            details = f"\n{exc.response.text[:1000]}"
        raise SystemExit(f"CometAPI request failed: {exc}{details}") from exc
    except ValueError as exc:
        raise SystemExit("CometAPI returned invalid JSON.") from exc

    print(extract_text(payload))

    usage = payload.get("usageMetadata")
    if usage:
        print(f"\nUsage: {usage}")


if __name__ == "__main__":
    main()
```

## Privacy and consent

Audio can contain personal, confidential, or legally protected information. Before uploading a recording:

- Obtain any consent required by applicable law.
- Confirm that you are authorized to share it with CometAPI.
- Review CometAPI’s privacy, retention, and data-handling terms.
- Avoid uploading secrets or regulated data unless permitted.
- Follow your organization’s deletion and retention policies.

This project is not legal advice.

## Accuracy limitations

Treat every result as a draft:

- Verify quotations word for word.
- Check timestamps against the recording.
- Confirm speaker identities manually.
- Review names, dates, amounts, and commitments.
- Expect lower accuracy with noise, crosstalk, music, or unclear speech.
- Never treat an extracted action item as authorization for a consequential action.

## Large files

Base64 encoding increases request size by roughly one-third. Large recordings may exceed the gateway’s inline-upload or request limits.

If that happens, consider:

- Using a supported compressed audio format
- Reducing unnecessary bitrate or channels
- Splitting the recording into clearly labeled segments
- Using a provider-supported file-upload workflow, if available

## Compatibility

Before use, confirm that CometAPI currently supports:

- The `gemini-4-argon` model identifier
- Audio input for that deployment
- The Gemini-compatible `generateContent` endpoint
- Your audio format and request size
- The request and response fields shown above
- Current pricing and rate limits

## Links

- [Gemini 4 Argon on CometAPI](https://www.cometapi.com/models/google/gemini-4-argon/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=gemini-4-argon)
- [CometAPI documentation](https://apidoc.cometapi.com)
