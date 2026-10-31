# Base44 sandbox notes

Non-obvious things about running this app in the Base44 sandbox. Everyday
project usage is covered by `README.md`.

## Running it here

- `docker compose -f docker-compose.base44.yml up -d --build` (web entry on
  host port 3000 → container port 8000).
- `Dockerfile.base44` builds a dev image: source is bind-mounted, uvicorn runs
  with `--reload`. Torch is installed from the PyTorch **CPU** wheel index
  (the default PyPI build drags in ~1.5 GB of CUDA libraries and this service
  runs with `gpu=False`).
- `requirements.txt` pins `numpy==1.24.3`, so scipy/scikit-image are held to
  `scipy<1.12` / `scikit-image<0.22` via a constraints file inside the image
  build. Loosen that only together with the numpy pin.
- `pdf2image` needs `poppler-utils` in the image (already installed).

## Startup is slow by design

The FastAPI startup hook builds the EasyOCR reader, which downloads
`craft_mlt_25k.pth` (~83 MB) and `english_g2.pth` (~15 MB) before uvicorn
accepts requests. Models live in the `easyocr-models` named volume mounted at
`/app/models`, so they survive restarts; the healthcheck's `start_period` is
300s to cover a cold download. Do not mount `models/` from the repo instead —
writes there would land in git (only `*.pth` is ignored).

## Verifying it works

- `curl localhost:3000/health` → `{"status":"healthy",...}`
- `curl localhost:3000/` → API info JSON
- Swagger UI: `/docs` (this is the only browsable UI; `/` is JSON)
- End-to-end OCR check:

```bash
docker compose -f docker-compose.base44.yml exec -T api python - <<'PY'
from PIL import Image, ImageDraw
img = Image.new("RGB", (900, 400), "white")
d = ImageDraw.Draw(img)
d.text((40, 40), "MD: ZAKIR HOSSAIN", fill="black")
d.text((40, 180), "NID: 1234567890123", fill="black")
img.save("/tmp/nid_sample.png")
PY
docker compose -f docker-compose.base44.yml exec -T api \
  curl -s -X POST http://127.0.0.1:8000/extract-nid-info/ \
  -F "type=NID" -F "file=@/tmp/nid_sample.png;type=image/png"
```

`POST /extract-nid-info/` requires **both** the `type` form field (NID|BO|TIN)
and the `file` field — omitting `type` returns 422.

## Sandbox-only configuration

- **`GET /` landing page.** The Base44 preview panel needs an HTML document at
  the home route; a JSON body leaves it showing nothing useful. When
  `BASE44_PREVIEW_MODE` is exactly `"1"`, `root()` returns
  `static/index.html` (a thin shell that embeds the Swagger UI from `/docs`).
  When the variable is unset or any other value, `/` returns the original
  documented JSON API-info payload — `test_api.py` and the README contract are
  untouched. `static/index.html` is the only piece of UI in this repo.
- `BASE44_PREVIEW_MODE` is otherwise unused: the app has no host/origin
  allowlist to extend (CORS is `*`, it binds `0.0.0.0:8000`).
- No external credentials are required: OCR runs fully locally.
