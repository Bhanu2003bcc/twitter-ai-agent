# Twitter AI Agent

A production-grade AI agent that generates Twitter-ready posts with AI-generated images and auto-publishes to your Twitter/X account — with a confirmation step before posting.

```
User Input → FastAPI → Gemini (text) →  Pollinations AI (image) → Review UI → Twitter API
```

---

## Features

- **AI Text Generation** — Gemini 2.5 Flash generates Twitter posts, captions, and hashtags
- **AI Image Generation** — HuggingFace Stable Diffusion creates topic-relevant images
- **Confirmation Step** — Review everything before it goes live
- **Web UI** — Clean browser interface served by FastAPI
- **CLI Mode** — Run the full agent from your terminal
- **Session Management** — Redis or in-memory caching for workflow state
- **REST API** — Full `/generate` → `/confirm` → posted workflow
- **Auto Fallback** — Tries backup image models if primary fails
- **Free Tier APIs** — Uses Gemini free tier and HuggingFace free inference

---

## Project Structure

```
twitter-ai-agent/
├── app/
│   ├── main.py                  # FastAPI app entry point
│   ├── api/
│   │   └── routes.py            # All API endpoints
│   ├── core/
│   │   ├── agent.py             #  Master agent orchestrator
│   │   ├── config.py            # Settings & env variables
│   │   └── logger.py            # Logging setup
│   ├── models/
│   │   └── schemas.py           # Pydantic data models
│   └── services/
│       ├── gemini_service.py    # Google Gemini API (text)
│       ├── huggingface_service.py # HuggingFace API (images)
│       ├── linkedIn_service.py   # Linked In API (posting)
│       └── cache_service.py     # Redis / in-memory cache
├── frontend/
│   └── index.html               # Web UI
├── tests/
│   └── test_agent.py            # Unit + integration tests
├── cli.py                       # Terminal CLI interface
├── requirements.txt
├── .env.example                 # ← Copy this to .env
└── README.md
```

---

## Quick Start

### 1. Clone & Install

```bash
git clone <your-repo-url>
cd twitter-ai-agent

# Create virtual environment
python -m venv venv
source venv/bin/activate        # Linux/Mac
# venv\Scripts\activate         # Windows

# Install dependencies
pip install -r requirements.txt
```

### 2. Configure API Keys

```bash
cp .env.example .env
```

Open `.env` and fill in your keys:

```env
GEMINI_API_KEY=your_key_here
HUGGINGFACE_API_KEY=hf_your_token_here

```

### 3. Run the Server

```bash
python -m app.main
```

Then open **http://localhost:8000** in your browser.

---

## Getting API Keys

### Gemini API (Free)

1. Go to [https://aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey)
2. Click **Create API Key**
3. Copy it into `.env` as `GEMINI_API_KEY`

> Free tier: 15 requests/minute, 1 million tokens/day — plenty for personal use.

---

### HuggingFace API (Free)

1. Sign up at [https://huggingface.co](https://huggingface.co)
2. Go to **Settings → Access Tokens**
3. Click **New Token** → select **Read** role
4. Copy it into `.env` as `HUGGINGFACE_API_KEY`

> Free tier: Serverless inference on thousands of models including Stable Diffusion.
>
> First request may take 20–40s if the model is "cold" (loading). Subsequent requests are faster.

# ── HuggingFace — NOT NEEDED ANYMORE ──────────────────────
# Using Pollinations AI instead (free, no key required)
# HUGGINGFACE_API_KEY=not_needed
# HUGGINGFACE_IMAGE_MODEL=not_needed

---

### Twitter / X API

You need a **Developer Account** with **Read + Write** permissions.

1. Go to [https://developer.twitter.com/en/portal/dashboard](https://developer.twitter.com/en/portal/dashboard)
2. Create a new **Project** and **App**
3. Under **App Settings → User authentication settings**:
   - Enable **OAuth 1.0a**
   - Set permissions to **Read and Write**
   - Set callback URL to `http://localhost:8000/callback`
4. Under **Keys and Tokens**:
   - Copy **API Key** and **API Key Secret**
   - Click **Generate** under Access Token and Secret
   - Copy both into your `.env`

> Twitter's free tier (Basic) allows **1,500 tweets/month**. The agent respects this.

---

## Using the Web UI

1. Start the server: `python -m app.main`
2. Open **http://localhost:8000**
3. Type your topic in the text area
4. Adjust tone, hashtags, emoji options
5. Click ** Generate Content**
6. Review the tweet preview and AI image
7. Click ** Post to Linked In** — or **Cancel**

---

## Using the CLI

```bash
# Interactive mode
python cli.py

# With input
python cli.py --input "Today I learned about Apache Kafka"

# With tone
python cli.py --input "Just shipped a new feature!" --tone casual

# Available tones
python cli.py --input "ML fundamentals explained" --tone educational
```

**CLI Tones:** `professional` | `casual` | `enthusiastic` | `educational`

---

## REST API Reference

The agent exposes a clean REST API. Useful for integrations or automation.

### POST `/api/v1/generate`

Generate LinkedIn post + image for a given input.

```bash
curl -X POST http://localhost:8000/api/v1/generate \
  -H "Content-Type: application/json" \
  -d '{
    "user_input": "Today I learned about Apache Kafka",
    "include_hashtags": true,
    "include_emojis": true,
    "tone": "professional"
  }'
```

**Response:**
```json
{
  "success": true,
  "session_id": "abc123-...",
  "status": "ready",
  "content": {
    "twitter_post": " Just deep-dived into Apache Kafka...",
    "caption": "Kafka is a distributed event streaming platform...",
    "image_prompt": "Abstract visualization of data streams...",
    "image_url": "/api/v1/images/post_a1b2c3.png",
    "image_base64": "data:image/png;base64,...",
    "hashtags": ["#Kafka", "#DataEngineering", "#Tech"],
    "topic": "Apache Kafka",
    "char_count": 187
  },
  "processing_time_ms": 8432
}
```

---

### POST `/api/v1/confirm`

Confirm posting (or cancel) using the `session_id` from generate.

```bash
# Post to Twitter
curl -X POST http://localhost:8000/api/v1/confirm \
  -H "Content-Type: application/json" \
  -d '{"session_id": "abc123-...", "confirmed": true}'

# Cancel
curl -X POST http://localhost:8000/api/v1/confirm \
  -H "Content-Type: application/json" \
  -d '{"session_id": "abc123-...", "confirmed": false}'
```

**Response:**
```json
{
  "success": true,
  "session_id": "abc123-...",
  "status": "posted",
  "post_id": "1234567890",
  "post_url": "https://twitter.com/i/web/status/1234567890",
  "message": " Posted to Twitter!",
  "posted_at": "2024-01-15T12:34:56"
}
```

---

### GET `/api/v1/session/{session_id}`

Check status of a session.

```bash
curl http://localhost:8000/api/v1/session/abc123-...
```

**Session statuses:** `pending` → `processing` → `ready` → `posted` / `failed` / `cancelled`

---

### GET `/api/v1/status`

Check which services are configured.

```bash
curl http://localhost:8000/api/v1/status
```

---

### GET `/api/v1/images/{filename}`

Serve a generated image.

```bash
curl http://localhost:8000/api/v1/images/post_a1b2c3.png --output image.png
```

---

## 🧪 Running Tests

```bash
# Unit tests only (no API keys needed)
pytest tests/ -v

# Include integration tests (requires API keys in .env)
pytest tests/ -v -m integration

# With coverage
pip install pytest-cov
pytest tests/ -v --cov=app --cov-report=html
```

---

## ⚙️ Configuration Reference

All settings live in `.env`. Here's the full reference:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `GEMINI_API_KEY` | ✅ Yes | — | Google Gemini API key |
| `GEMINI_MODEL` | No | `gemini-1.5-flash` | Gemini model to use |
| `HUGGINGFACE_API_KEY` | ✅ Yes | — | HuggingFace access token |
| `HUGGINGFACE_IMAGE_MODEL` | No | `stabilityai/stable-diffusion-xl-base-1.0` | Image generation model |
| `REDIS_ENABLED` | No | `false` | Enable Redis caching |
| `REDIS_URL` | No | `redis://localhost:6379` | Redis connection URL |
| `IMAGE_SAVE_DIR` | No | `generated_images` | Where to save images |
| `API_TIMEOUT` | No | `60` | HTTP request timeout (seconds) |
| `MAX_RETRIES` | No | `3` | API retry attempts |
| `ENVIRONMENT` | No | `development` | `development` or `production` |

---

## Agent Workflow Explained

```
┌─────────────────────────────────────────────────────────┐
│                    AGENT WORKFLOW                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. INPUT          User submits topic/text               │
│       ↓                                                  │
│  2. PROMPT BUILD   Constructs structured Gemini prompt   │
│       ↓                                                  │
│  3. GEMINI API     Returns: post text, caption,          │
│                    image prompt, hashtags, topic         │
│       ↓                                                  │
│  4. HUGGINGFACE    Generates image from image_prompt     │
│       ↓                                                  │
│  5. COMBINE        Merges text + image into session      │
│       ↓                                                  │
│  6. REVIEW         Shows user the generated content      │
│       ↓                                                  │
│  7. CONFIRM        User clicks OK / Cancel               │
│       ↓                                                  │
│  8. Linked In        Uploads image → posts               │
│       ↓                                                  │
│  9. DONE           Returns tweet URL                     │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**Session States:**

```
pending → processing → ready → posted
                    ↘         ↗
                     failed
                    ↘
                     cancelled
```

---

## 🐳 Docker (Optional)

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
# Build and run
docker build -t twitter-ai-agent .
docker run -p 8000:8000 --env-file .env twitter-ai-agent
```

---

## 🛠️ Troubleshooting

### Image generation is slow / times out

HuggingFace free tier loads models on demand. First request may take 30–60 seconds. Try a faster model:
```env
HUGGINGFACE_IMAGE_MODEL=runwayml/stable-diffusion-v1-5
```

### "Model is currently loading" error

This is normal for cold starts on HuggingFace free tier. The agent automatically retries. If it persists, wait a minute and try again.

### Twitter API returns 403

Ensure your Twitter App has **Read and Write** permissions enabled under *User authentication settings*, and that you regenerated your Access Token **after** changing permissions.

### Gemini returns rate limit (429)

The free tier allows 15 RPM. The agent retries with exponential backoff automatically.

### Redis connection refused

If you don't have Redis, set `REDIS_ENABLED=false` in `.env`. The agent uses in-memory caching as a fallback — sessions will reset when the server restarts.

---

## 📦 Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend API | FastAPI + Uvicorn |
| Text Generation | Google Gemini 1.5 Flash |
| Image Generation | HuggingFace Stable Diffusion |
| Linked In Posting | Twitter API v2 + OAuth 1.0a |
| Data Models | Pydantic v2 |
| Async HTTP | httpx + asyncio |
| Caching | Redis (or in-memory fallback) |
| Frontend | Vanilla HTML/CSS/JS |
| Testing | pytest + pytest-asyncio |

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

## 🙋 FAQ

**Q: Can I use this without Linked In credentials?**
Yes! The generate step works without linked In keys. You'll see the generated post and image, but the "Post to Linked" button will show an error until credentials are configured.

**Q: Is this free to use?**
The Gemini and HuggingFace APIs have generous free tiers sufficient for personal use. Twitter's Basic plan ($100/month) is required for the API, but the free developer tier allows limited posting for testing.

**Q: Can I change the AI model?**
Yes — update `GEMINI_MODEL` and `HUGGINGFACE_IMAGE_MODEL` in `.env`. Any HuggingFace text-to-image model that supports the Inference API will work.

**Q: How do I run this in production?**
Set `ENVIRONMENT=production`, use a proper `SECRET_KEY`, enable Redis (`REDIS_ENABLED=true`), and run behind a reverse proxy (nginx). Consider adding rate limiting middleware.
# twitter-ai-agent
