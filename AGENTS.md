# RedInk AGENT Notes

Internal map of the repo so an AI can safely extend it.

## Quick Run
- Backend: `uv sync` → `uv run python -m backend.app` (serves /api, and / if `frontend/dist` exists).
- Frontend: `cd frontend && pnpm install && pnpm dev`.
- Docker: `docker-compose up -d` (persists `history/` and `output/`).

## Backend (Flask, Python 3.11)
- Entry: `backend/app.py` builds Flask with CORS, serves built frontend when `frontend/dist` exists, and registers blueprints via `backend.routes`.
- Config: `backend/config.py` loads `image_providers.yaml` and `text_providers.yaml` (cached). `Config.get_image_provider_config` validates api_key/base_url/model. `Config.reload_config` clears caches (used after POST /api/config).
- Routes (all prefixed with /api):
  - `outline_routes.py`: `POST /outline` -> OutlineService, supports multipart uploads or base64 images.
  - `image_routes.py`: `POST /generate` streams SSE events `progress|complete|error|finish`; `GET /images/<task>/<file>` with optional `thumbnail`; `POST /retry`, `POST /retry-failed` (SSE), `POST /regenerate`, `GET /task/<id>`, `GET /health`.
  - `history_routes.py`: CRUD on JSON records in `history/`, search, stats, scan/sync tasks, and `GET /history/<id>/download` zip.
  - `config_routes.py`: `GET/POST /config` to read/write provider YAML (API keys masked on read), `POST /config/test` to probe providers.
- Services:
  - `backend/services/outline.py` OutlineService loads `text_providers.yaml`, picks active provider, uses `backend.utils.text_client.get_text_chat_client` to call Gemini/OpenAI-compatible chat with optional images, prompt from `backend/prompts/outline_prompt.txt`, and parses pages split by `<page>`.
  - `backend/services/image.py` ImageService keeps per-task state in memory + filesystem (`history/<task_id>/`). Uses `ImageGeneratorFactory` to create provider client, supports short/long prompts (`backend/prompts/image_prompt[_short].txt`), compresses refs, auto-retries, and streams SSE. Covers: cover-first generation, optional high_concurrency via ThreadPoolExecutor, thumbnails via `image_compressor.py`, retry/regenerate APIs honoring stored context (full_outline, cover image, user images/topic).
  - `backend/services/history.py` stores records as JSON files and an `index.json` under `history/`. Tracks status draft/generating/completed/partial, thumbnails, task_id; `scan_and_sync` scans disk task folders to reconcile missing metadata.
- Generators (`backend/generators`):
  - Factory: `ImageGeneratorFactory` registers `google_genai`, `openai_compatible|openai`, `image_api`.
  - `google_genai.py`: Google Generative AI image streaming with safety settings off; supports reference images; detailed error parsing for common 4xx/5xx/safety/rate issues.
  - `openai_compatible.py`: OpenAI-style images or chat endpoints; handles b64_json/url responses, rate-limit retries.
  - `image_api.py`: Generic vendor; supports `/v1/images/generations` or chat-like endpoints, multi-reference images, compresses refs; retries with backoff.
- Utils:
  - `utils/text_client.py`: OpenAI-compatible chat client with optional images (base64 data URLs) and 429 retry.
  - `utils/genai_client.py`: Gemini client (text+image) with smart retry + `parse_genai_error`.
  - `utils/image_compressor.py`: Pillow-based size clamp (KB + resize) used before uploads/thumbnails.

### Data & Storage
- Tasks: images saved to `history/<task_id>/<index>.png` plus `thumb_<index>.png`.
- History records: `history/<record_id>.json` and `history/index.json` summary. Download zip bundles via `/api/history/<id>/download` (skips thumbnails, names page_N.png).
- Output dir default `output/` unused in code now but created in Docker image.

### Prompts
- Outline prompt: `backend/prompts/outline_prompt.txt` enforces `<page>` separators and `[封面]/[内容]/[总结]` tags.
- Image prompt: `backend/prompts/image_prompt.txt` (full context + user topic + outline, style rules, compliance notes), short variant for providers set with `short_prompt: true`.

## Frontend (Vue 3 + TS + Vite + Pinia)
- Entry: `frontend/src/main.ts`, global styles under `frontend/src/assets/css`.
- Router (`router/index.ts`): `/` home, `/outline`, `/generate`, `/result`, `/history`, `/history/:id`, `/settings`.
- State: `stores/generator.ts` holds `topic`, `outline.raw/pages`, `images` with statuses, `taskId`, `recordId`, `userImages`; stages input→outline→generating→result; auto-persists to localStorage via `setupAutoSave`.
- API client: `api/index.ts` wraps axios/fetch. Key flows:
  - `generateOutline(topic, images?)` (multipart if files).
  - `generateImagesPost(pages, taskId, fullOutline, …)` listens to SSE events progress/complete/error/finish.
  - Retry helpers `regenerateImage`, `retryFailedImages`, history CRUD/search/stats/scan, config read/write/test, image URL helper `getImageUrl(taskId, filename, thumbnail)`.
- Views:
  - `HomeView.vue` (with `components/home/ComposerInput.vue`, `ShowcaseBackground.vue`): captures topic + optional images, calls outline API, routes to outline.
  - `OutlineView.vue`: edit/drag/delete/add pages; starts generation.
  - `GenerateView.vue`: consumes SSE stream, updates progress, auto-creates history via `createHistory`, calls `updateHistory` on finish, supports single/batch retry/regenerate with context (full_outline/user_topic), routes to result on success.
  - `ResultView.vue`: simple gallery/download CTA.
  - `HistoryView.vue`: lists records with tabs/search/stats (`components/history/StatsOverview.vue`), cards (`GalleryCard.vue`), viewer modal (`ImageGalleryModal.vue`) with per-image regenerate/download and outline modal; triggers backend scan-all for sync.
  - `SettingsView.vue`: manages provider configs via `ProviderTable.vue` + modals (`ProviderModal.vue`, `ImageProviderModal.vue`), calls `/api/config` and `/api/config/test`.

## Extending Safely
- Adding an image provider: implement `ImageGeneratorBase`, register via `ImageGeneratorFactory.register_generator`, ensure `image_providers.yaml` includes `type` matching registration and required fields (api_key, base_url/model). If API differs, update `/api/config/test` with a new branch.
- Tweaking prompts or styles: edit files under `backend/prompts/`; keep required placeholders (`{page_content}`, `{page_type}`, `{full_outline}`, `{user_topic}`) intact or adjust `ImageService` formatting.
- Changing SSE contract: frontend expects `event:` lines `progress|complete|error|finish` (and `retry_start|retry_finish` for batch retry) with JSON `data` containing `index/status/image_url/message/...`. Update both generator stream parsers and UI handlers together.
- History model: stored fields (`id,title,outline,images{task_id,generated[]},status,thumbnail,created_at,updated_at`) are consumed by HistoryView/Gallery components; maintain shape when extending.
- Static hosting: when bundling frontend, place build output in `frontend/dist` so Flask serves `/` and history mode fallback.

## Useful Paths
- Backend app: `backend/app.py`, routes under `backend/routes/`.
- Services: `backend/services/outline.py`, `backend/services/image.py`, `backend/services/history.py`.
- Generators: `backend/generators/`.
- Frontend app shell: `frontend/src/App.vue`.
- State: `frontend/src/stores/generator.ts`.
- API wrapper: `frontend/src/api/index.ts`.

Keep edits ASCII, avoid removing user data under `history/` or unrelated worktree changes.***
