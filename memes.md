
## 🎯 Goal  
While you wait for Reddit‑API (PRAW) credentials, let the **scraper** keep feeding the pipeline with fresh memes from **other public sources**.  
We’ll:

1. **Add a “source plug‑in system** to `scraper/` so you can switch between any number of providers (Reddit, X/Twitter, 9GAG, Meme‑API, a local folder, …).  
2. **Extend `config.yaml`** with a list of enabled sources and the credentials each source needs.  
3. **Provide ready‑to‑run Python modules** for the most common free meme sources.  
4. **Keep the same output contract** (`/data/raw/<filename>` + a JSON manifest) so the rest of the pipeline (scoring → AI‑mod) stays untouched.  

You can enable one source, several sources, or all of them at once – the scraper will de‑duplicate by image URL.

---

## 1️⃣ Updated `config.yaml`

Add a top‑level `sources` block that tells the scraper which providers to run and where their credentials live.

```yaml
# -------------------------------------------------
# Global configuration – edit the placeholders
# -------------------------------------------------
aws:
  region: us-east-1
  raw_bucket: meme-raw
  filtered_bucket: meme-filtered
  sqs_queue_url: https://sqs.us-east-1.amazonaws.com/123456789012/meme-mod-queue

local:
  endpoint: http://localhost:9000
  access_key: minioadmin
  secret_key: minioadmin
  raw_bucket: meme-raw
  filtered_bucket: meme-filtered
  sqs_queue_url: http://localhost:9324/queue/meme-mod-queue

reddit:
  client_id: YOUR_REDDIT_CLIENT_ID
  client_secret: YOUR_REDDIT_CLIENT_SECRET
  user_agent: meme-automation-bot

twitter:
  bearer_token: YOUR_TWITTER_BEARER_TOKEN   # for v2 API (read‑only)

meme_api:
  base_url: https://meme-api.com/gimme   # public, no auth needed

sources:
  # Order matters – the scraper will call them sequentially.
  # Set `enabled: true` for any you want to run now.
  - name: reddit
    enabled: false          # will be turned on once you have the credentials
    subreddits:
      - memes
      - dankmemes
  - name: twitter
    enabled: true
    hashtags:
      - "#meme"
      - "#dankmemes"
    max_per_hashtag: 5
  - name: ninegag
    enabled: true
    max_posts: 10
  - name: meme_api
    enabled: true
    max_posts: 8
  - name: local_folder
    enabled: true
    path: ./sample_memes          # any folder on the host with .jpg/.png files
```

> **Tip:** You can add more providers later – just create a new Python module under `scraper/providers/` and reference it in the `sources` list.

---

## 2️⃣ Directory layout (new files)

```
meme‑merch‑automation/
├─ scraper/
│   ├─ providers/
│   │   ├─ __init__.py
│   │   ├─ reddit.py          # unchanged – still there for when you get credentials
│   │   ├─ twitter.py
│   │   ├─ ninegag.py
│   │   ├─ meme_api.py
│   │   └─ local_folder.py
│   ├─ requirements.txt
│   ├─ scraper.py            # orchestrator – now loads providers dynamically
│   └─ entrypoint.sh
```

---

## 3️⃣ Provider interface (contract)

Every provider module must expose **one function**:

```python
def fetch(limit: int = 10) -> List[Dict]:
    """
    Return a list of dicts, each dict having at least:
        {
            "id":   <unique‑string‑id>,
            "title": <optional‑title>,
            "url":   <direct‑link‑to‑image‑file>,
            "score": <numeric‑popularity‑proxy (e.g. retweets, upvotes)>,
            "created_utc": <unix‑timestamp>,
            "source": "<provider‑name>"
        }
    The orchestrator will download the image, add a local_path entry,
    and later write the manifest.
    """
```

All providers **must not** write files themselves – the orchestrator handles downloading and S3 upload.  
If a provider cannot retrieve an image (e.g., broken URL), it should simply **skip** that entry.

---

## 4️⃣ Provider implementations

> **All of these modules use only the libraries already listed in `scraper/requirements.txt`.**  
> If you add a new library, bump the `requirements.txt` accordingly.

### 4.1 `scraper/providers/__init__.py`

```python
# Empty – just makes the folder a package.
```

### 4.2 `scraper/providers/twitter.py`

```python
#!/usr/bin/env python3
"""
Twitter (X) provider – uses the v2 recent search endpoint.
Requires a Bearer Token with read‑only access.
"""

import os, time, logging, requests
from typing import List, Dict

log = logging.getLogger(__name__)

BEARER_TOKEN = os.getenv("TWITTER_BEARER_TOKEN")
BASE_URL = "https://api.twitter.com/2/tweets/search/recent"

def _auth_headers():
    return {"Authorization": f"Bearer {BEARER_TOKEN}"}

def fetch(limit: int = 5, hashtags: List[str] = None) -> List[Dict]:
    """
    Pull the most recent tweets that contain any of the supplied hashtags.
    Returns a list of meme‑like entries (image URLs only).
    """
    if not BEARER_TOKEN:
        log.warning("Twitter bearer token not set – skipping Twitter source")
        return []

    if not hashtags:
        hashtags = ["#meme"]

    results = []
    for tag in hashtags:
        query = f"{tag} has:images -is:retweet lang:en"
        params = {
            "query": query,
            "max_results": min(limit, 10),   # API caps at 10 per request
            "tweet.fields": "created_at,public_metrics",
            "expansions": "attachments.media_keys",
            "media.fields": "url,preview_image_url"
        }
        resp = requests.get(BASE_URL, headers=_auth_headers(), params=params, timeout=10)
        if resp.status_code != 200:
            log.error(f"Twitter API error {resp.status_code}: {resp.text}")
            continue
        data = resp.json()
        tweets = data.get("data", [])
        media_map = {m["media_key"]: m for m in data.get("includes", {}).get("media", [])}

        for tw in tweets:
            # Find the first image media attached to the tweet
            media_keys = tw.get("attachments", {}).get("media_keys", [])
            img_url = None
            for mk in media_keys:
                media = media_map.get(mk, {})
                img_url = media.get("url") or media.get("preview_image_url")
                if img_url:
                    break
            if not img_url:
                continue

            entry = {
                "id": f"twitter_{tw['id']}",
                "title": tw.get("text", "").replace("\n", " ").strip(),
                "url": img_url,
                "score": tw.get("public_metrics", {}).get("like_count", 0),
                "created_utc": int(time.mktime(time.strptime(tw["created_at"], "%Y-%m-%dT%H:%M:%S.%fZ"))),
                "source": "twitter"
            }
            results.append(entry)

        # Stop early if we already hit the global limit
        if len(results) >= limit:
            break

    return results[:limit]
```

### 4.3 `scraper/providers/ninegag.py`

```python
#!/usr/bin/env python3
"""
9GAG provider – scrapes the public “hot” page.
No API key required; just a simple HTML parse.
"""

import logging, requests, time, uuid
from bs4 import BeautifulSoup
from typing import List, Dict

log = logging.getLogger(__name__)

BASE_URL = "https://9gag.com/hot"

def fetch(limit: int = 10) -> List[Dict]:
    resp = requests.get(BASE_URL, timeout=10)
    if resp.status_code != 200:
        log.error(f"9GAG request failed: {resp.status_code}")
        return []

    soup = BeautifulSoup(resp.text, "html.parser")
    posts = soup.select("article[data-entry-id]")[:limit]

    results = []
    for post in posts:
        img_tag = post.select_one("img[data-src]")
        if not img_tag:
            continue
        img_url = img_tag["data-src"]
        # 9GAG does not expose a numeric score; we fake one with a random int
        entry = {
            "id": f"ninegag_{post['data-entry-id']}",
            "title": post.get("data-title", ""),
            "url": img_url,
            "score": 0,                     # placeholder – scoring will treat it as low
            "created_utc": int(time.time()),  # use now as creation time
            "source": "ninegag"
        }
        results.append(entry)

    return results
```

> **Dependency:** `beautifulsoup4` – add it to `scraper/requirements.txt`:

```text
beautifulsoup4==4.12.3
```

### 4.4 `scraper/providers/meme_api.py`

```python
#!/usr/bin/env python3
"""
Meme‑API.com – a public, no‑auth endpoint that returns a random meme.
We call it repeatedly until we hit the requested limit.
"""

import os, logging, requests, time, uuid
from typing import List, Dict

log = logging.getLogger(__name__)

BASE_URL = os.getenv("MEME_API_BASE_URL", "https://meme-api.com/gimme")

def fetch(limit: int = 8) -> List[Dict]:
    results = []
    attempts = 0
    while len(results) < limit and attempts < limit * 3:
        attempts += 1
        resp = requests.get(BASE_URL, timeout=10)
        if resp.status_code != 200:
            log.warning(f"Meme‑API request failed ({resp.status_code})")
            continue
        data = resp.json()
        # The API returns a single meme per call:
        #   {"postLink":"...","title":"...","url":"...","nsfw":false,"spoiler":false}
        if data.get("nsfw") or data.get("spoiler"):
            continue
        entry = {
            "id": f"memeapi_{uuid.uuid4().hex}",
            "title": data.get("title", ""),
            "url": data.get("url"),
            "score": 0,                     # no popularity metric – treat as neutral
            "created_utc": int(time.time()),
            "source": "meme_api"
        }
        results.append(entry)
    return results
```

### 4.5 `scraper/providers/local_folder.py`

```python
#!/usr/bin/env python3
"""
Local folder provider – useful for testing or when you have a curated set of images.
"""

import os, time, uuid, logging
from pathlib import Path
from typing import List, Dict

log = logging.getLogger(__name__)

def fetch(limit: int = 20, path: str = "./sample_memes") -> List[Dict]:
    folder = Path(path)
    if not folder.is_dir():
        log.error(f"Local folder {path} does not exist")
        return []

    images = list(folder.glob("*.jpg")) + list(folder.glob("*.jpeg")) + list(folder.glob("*.png"))
    images = images[:limit]

    results = []
    for img_path in images:
        entry = {
            "id": f"local_{uuid.uuid4().hex}",
            "title": img_path.stem,
            "url": f"file://{img_path.resolve()}",
            "score": 0,
            "created_utc": int(time.time()),
            "source": "local_folder"
        }
        results.append(entry)

    return results
```

---

## 5️⃣ Orchestrator – `scraper/scraper.py` (updated)

```python
#!/usr/bin/env python3
"""
Unified meme scraper – pulls memes from any enabled provider(s) defined in config.yaml.
The output contract (images + manifest) is identical to the original Reddit‑only version,
so downstream steps (scoring, AI‑mod, etc.) stay unchanged.
"""

import os, json, datetime, pathlib, logging, boto3, requests, uuid, time
from pathlib import Path
import importlib
import yaml   # pip install pyyaml (already in requirements)

log = logging.getLogger()
log.setLevel(logging.INFO)

# -----------------------------------------------------------------
# Load configuration (mounted as /app/config.yaml)
# -----------------------------------------------------------------
CONFIG_PATH = "/app/config.yaml"
with open(CONFIG_PATH) as f:
    cfg = yaml.safe_load(f)

# -----------------------------------------------------------------
# Helper: download an image (supports http(s) and file:// URLs)
# -----------------------------------------------------------------
def download_image(url: str, dest: Path):
    if url.startswith("file://"):
        # Local file – just copy
        src = Path(url.replace("file://", ""))
        dest.parent.mkdir(parents=True, exist_ok=True)
        dest.write_bytes(src.read_bytes())
        return

    r = requests.get(url, stream=True, timeout=10)
    r.raise_for_status()
    dest.parent.mkdir(parents=True, exist_ok=True)
    with open(dest, "wb") as f:
        for chunk in r.iter_content(1024):
            f.write(chunk)

# -----------------------------------------------------------------
# Load provider modules dynamically based on config.yaml
# -----------------------------------------------------------------
def load_providers():
    providers = []
    for src_cfg in cfg.get("sources", []):
        if not src_cfg.get("enabled", False):
            continue
        name = src_cfg["name"]
        try:
            mod = importlib.import_module(f"scraper.providers.{name}")
        except ImportError as e:
            log.error(f"Could not import provider '{name}': {e}")
            continue

        # Build a small wrapper that calls the provider with its own args
        def make_fetch(mod, src_cfg):
            def fetch():
                # Most providers accept `limit` and maybe extra kwargs
                limit = src_cfg.get("max_posts") or src_cfg.get("max_per_hashtag") or 10
                # Pass provider‑specific kwargs (e.g. hashtags, path, subreddits)
                kwargs = {k: v for k, v in src_cfg.items()
                          if k not in {"name", "enabled", "max_posts", "max_per_hashtag"}}
                return mod.fetch(limit=limit, **kwargs)
            return fetch

        providers.append(make_fetch(mod, src_cfg))
        log.info(f"Enabled provider: {name}")
    return providers

# -----------------------------------------------------------------
# Main routine
# -----------------------------------------------------------------
def main():
    out_dir = Path("/data/raw")
    manifest = []

    providers = load_providers()
    if not providers:
        log.warning("No providers enabled – exiting.")
        return

    # -----------------------------------------------------------------
    # 1️⃣ Pull memes from each provider, de‑duplicate by URL
    # -----------------------------------------------------------------
    seen_urls = set()
    for fetch in providers:
        try:
            entries = fetch()
        except Exception as e:
            log.exception(f"Provider raised an exception: {e}")
            continue

        for entry in entries:
            url = entry["url"]
            if url in seen_urls:
                continue
            seen_urls.add(url)

            # -----------------------------------------------------------------
            # 2️⃣ Download image locally
            # -----------------------------------------------------------------
            img_name = f"{entry['source']}_{entry['id']}{Path(url).suffix}"
            local_path = out_dir / img_name
            try:
                download_image(url, local_path)
            except Exception as e:
                log.error(f"Failed to download {url}: {e}")
                continue

            entry["local_path"] = str(local_path)
            # Store a *s3* URL for later stages (optional)
            entry["s3_url"] = f"s3://{os.getenv('S3_BUCKET')}/{img_name}"
            manifest.append(entry)

    # -----------------------------------------------------------------
    # 3️⃣ Write manifest locally
    # -----------------------------------------------------------------
    ts = datetime.datetime.utcnow().strftime("%Y%m%d-%H%M%S")
    manifest_path = out_dir / f"metadata-{ts}.json"
    manifest_path.write_text(json.dumps(manifest, indent=2))
    log.info(f"Wrote manifest {manifest_path}")

    # -----------------------------------------------------------------
    # 4️⃣ Upload everything to S3 (or MinIO) – same code as before
    # -----------------------------------------------------------------
    s3 = boto3.client("s3", region_name=os.getenv("AWS_DEFAULT_REGION", "us-east-1"))
    bucket = os.getenv("S3_BUCKET")
    for entry in manifest:
        s3.upload_file(entry["local_path"], bucket,
                       Path(entry["local_path"]).name,
                       ExtraArgs={"ContentType": "image/jpeg"})
    s3.upload_file(str(manifest_path), bucket,
                   f"raw/{manifest_path.name}",
                   ExtraArgs={"ContentType": "application/json"})
    log.info(f"All files uploaded to s3://{bucket}/raw/")

if __name__ == "__main__":
    main()
```

**Key points**

* The script **reads the `sources` list** from `config.yaml`.  
* Each provider is imported on‑the‑fly (`scraper.providers.<name>`).  
* `limit` and any provider‑specific arguments are passed automatically.  
* Duplicate image URLs are ignored (helps when the same meme appears on multiple platforms).  
* The rest of the pipeline (scoring, AI‑mod) does **not** need any changes.

---

## 6️⃣ Updating `scraper/requirements.txt`

Add the new third‑party libraries:

```text
praw==7.7.1
requests==2.31.0
boto3==1.34.0
pyyaml==6.0.1
beautifulsoup4==4.12.3
```

If you later need `tweepy` (instead of raw `requests` for Twitter) you can add it, but the current implementation uses plain HTTP calls to avoid extra dependencies.

---

## 7️⃣ Quick test harness (run locally)

```bash
# 1️⃣ Build the Docker image (or just run locally with Python)
docker build -f docker/Dockerfile.scraper -t meme-scraper:latest .

# 2️⃣ Create a temporary .env that forces the k8s target (so the entrypoint loads it)
echo "DEPLOY_TARGET=k8s" > .env

# 3️⃣ Run the container locally, mounting a host folder for /data/raw
docker run --rm \
  -e DEPLOY_TARGET=k8s \
  -v "$(pwd)/sample_output:/data/raw" \
  -v "$(pwd)/config.yaml:/app/config.yaml" \
  -v "$(pwd)/.env:/app/.env" \
  meme-scraper:latest
```

*If you don’t have Docker handy, you can also run the script directly (it only needs the Python deps):*

```bash
pip install -r scraper/requirements.txt
python scraper/scraper.py
```

You should see a `metadata‑*.json` file in `sample_output/` and a handful of downloaded images.

---

## 8️⃣ What to do next

| Step | Action |
|------|--------|
| **Add your own API keys** | Fill `twitter.bearer_token` (or any other provider you enable) in `config.yaml`. |
| **Fine‑tune limits** | Adjust `max_posts` / `max_per_hashtag` per provider to stay within rate limits. |
| **Add more providers** | Create a new file under `scraper/providers/` that implements `fetch(limit, **kwargs)`. |
| **Test the whole pipeline** | Run `make build && make deploy-k8s` – the CronJobs will now pull memes from all enabled sources. |
| **When Reddit credentials arrive** | Flip `enabled: true` for the `reddit` block in `config.yaml`; the scraper will automatically start pulling from Reddit as well. |

You now have a **plug‑and‑play multi‑source meme collector** that works locally on MicroK8s and can later be pushed unchanged to AWS Lambda. Happy meme hunting! 🎉
