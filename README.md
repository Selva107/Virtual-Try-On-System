# Virtual Try-On Fashion System

Upload a photo, pick a garment (shirt, dress, saree, jacket…), see it on yourself.
Multi-user, role-based access control, async job pipeline, deployable with one command.

## 1. Run it

**Local (no Docker):**
```bash
pip install -r requirements.txt
uvicorn app.main:app --reload
# open http://localhost:8000   (API docs: /docs)
# default admin: admin@example.com / Admin@12345   (change it!)
```

**Production (Docker Compose: nginx + API + workers + Postgres + Redis):**
```bash
cp .env.example .env        # set SECRET_KEY, POSTGRES_PASSWORD, admin password
docker compose up -d --build
docker compose up -d --scale worker=4     # scale try-on capacity
```

**Tests:** `pytest -q tests`

## 2. Architecture

```
 Browser ──► nginx (rate limit, TLS, size cap)
                │
                ▼
        FastAPI  (N replicas, stateless)  ──►  Redis   (rate limits)
         │  auth · RBAC · quotas · validation
         │  enqueue job (row status=queued)
         ▼
     PostgreSQL ◄──── Workers (M replicas) claim jobs atomically
         ▲                │
         │                ▼
         │          Try-on engine  (overlay | your model via HTTP)
         ▼                │
   Shared storage  ◄──────┘   (volume now → S3/MinIO in cloud)
```

Key decisions
| Concern | Decision | Why |
|---|---|---|
| Heavy compute | Async job table + separate workers | API stays fast; scale workers independently |
| Job claiming | `UPDATE … WHERE status='queued'` (atomic) | no double-processing across workers |
| Stateless API | JWT, state in DB/Redis | add replicas behind the load balancer freely |
| Engine | `BaseEngine` interface | swap overlay ↔ your trained model without touching the API |
| Storage | Random filenames, authenticated download endpoints | photos are never publicly guessable |

## 3. Access control

Self-registration always creates role `user`. Only an admin can promote.

| Permission | user | premium | staff | admin |
|---|:-:|:-:|:-:|:-:|
| Browse catalog, create try-on, view own | ✅ | ✅ | ✅ | ✅ |
| High-quality output | – | ✅ | ✅ | ✅ |
| Add / deactivate garments | – | – | ✅ | ✅ |
| Manage users, view all jobs, stats, audit log | – | – | – | ✅ |
| Daily try-on quota | 10 | 100 | ∞ | ∞ |

Enforced server-side by `require("perm")` dependencies in `app/deps.py` (the UI only hides buttons).

Also: short-lived access tokens + refresh tokens; role/disable changes bump `token_version`
so existing tokens die immediately; `/api/auth/logout-all`; admins can't demote themselves;
other users' jobs return 404 (no id probing).

## 4. Security checklist
- PBKDF2-SHA256 (200k iterations) password hashing
- Upload validation: size cap, real image decode, JPEG/PNG/WEBP only, pixel cap
- Rate limiting on auth (Redis-backed, shared across replicas) + nginx limit
- Audit log of logins, role changes, uploads, deletions
- Users can delete their photo + result at any time (privacy)
- App refuses to boot in `ENV=prod` with the default `SECRET_KEY`
- Non-root container; secrets only via `.env`
- **Before going public:** enable HTTPS, rotate bootstrap admin password, restrict `CORS_ORIGINS`

## 5. Plug in your own model
Your existing mini-project model goes in `app/engine.py`:
```python
class MyEngine(BaseEngine):
    def run(self, person_path, garment_path, category, hq=False):
        ...            # load model, return a PIL.Image
```
Return it from `get_engine()`, or host the model separately and set
`TRYON_ENGINE=http` + `TRYON_HTTP_URL` (multipart `person`, `garment`, `category` → image bytes).
The built-in overlay engine is CPU-only (background cut-out, torso detection, blend). It is a
baseline, not photorealistic; for realistic drape use a diffusion VTON model through the HTTP engine.
Install `mediapipe` for better body detection.

## 5b. Live camera mode
The **Live camera** tab runs MediaPipe Pose *in the browser* (loaded from a CDN), so video
never leaves the user's device and the server is not loaded. Shoulders/hips/knees drive the
garment's position, size and rotation; the server only supplies the transparent garment PNG
(`GET /api/garments/{id}/cutout`). Needs a secure context: `http://localhost` works, any real
domain must use HTTPS, or the browser will block the camera.

How the live fit works (`frontend/fit.js`): the garment is split into a torso mesh and one mesh
per sleeve. Each sleeve is straightened into a tube and laid along shoulder → elbow → wrist, so
it bends with the elbow and matches arm length and thickness. Cloth lags slightly behind the
body (secondary motion), folds/armpit/elbow shading is computed from the mesh, brightness is
matched to the room, a soft contact shadow is added, and the user's hands, forearms and head
are drawn back in front of the cloth where they overlap it.
Limits: 2D warping of a flat photo; it cannot hide your original clothes underneath or show the
back of the garment. Photorealistic draping needs a diffusion/VTON model behind `HttpEngine`.

## 5c. Live try-on v2: single-piece fit, any outfit, body analysis, style suggestions

**One continuous garment (no detached sleeves).** `frontend/outfit.js` warps the whole garment photo as ONE mesh.
Tops, jackets and hoodies keep their sleeves attached to the body of the shirt; the sleeve part simply follows the
arm direction. Dresses, kurtas, sarees and gowns run shoulder → knee/ankle. **Bottoms** (pants, jeans, trousers,
shorts, skirt, leggings, palazzo) are mapped along the hip → knee → ankle skeleton of both legs; skirts ignore leg spread.
Pick a top *and* a bottom to wear both; tap a selected item again to take it off. A dress/kurta/saree is a full outfit.
If the catalog photo shows a model, the head/hair/skin are removed and the garment is scaled from the model's shoulders.

**Better body analysis** (`frontend/body.js`): One-Euro filtered landmarks (steady when still, quick when moving),
posture read-out (shoulder tilt, lean, facing direction, distance, what is in frame), optional *outline fit* (uses the
segmentation mask to measure your real chest/hip width so the garment matches your build), and size estimates
for **tops (XS–XXL) and bottoms (W28–W46)** corrected for body turn and using your saved scan's depth ratio. Pose model:
*full* on desktop, *lite* on phones.

**Body scan** (Live camera tab → "Full body scan"): enter your height, scan the front, turn sideways, scan the side.
The silhouette gives chest/waist/hip circumference (ellipse model, ±4–6 cm), saved per user
(`GET/PUT/DELETE /api/profile`, private).

**Style suggestions** (`GET /api/recommend?garment_ids=1,2&occasion=casual|work|party|ethnic|sport`, `app/recommender.py`):
works out the outfit's dominant colour (cached in `garments.color_hex`) and formality, then returns a colour palette
(neutrals, complementary, analogous), catalog garments that complete the outfit ranked by colour harmony, shoes,
accessories (watch metal tone, belt matched to shoes, jhumkas/bangles/dupatta for traditional wear, …) and tips.
Rule-based on purpose: explainable and no extra model to host. Accessories and shoes are suggestions (shown as
colour + type); only garments in your catalog can be tried on.

**AI photo try-on** (photorealistic, on the *Try on* tab): set `TRYON_ENGINE=hf` and
`pip install -r requirements-ai.txt`. It calls the public IDM-VTON Hugging Face Space (no local GPU; slow
at busy times, quota-limited unless `HF_TOKEN` is set, upper-body garments only). If it fails the job falls
back to the built-in overlay (`TRYON_FALLBACK=true`). For your own GPU box use `TRYON_ENGINE=http`.
The photo-upload tab handles tops/dresses; bottoms are fitted in the live camera tab.

Limits: it is still a 2D warp of a flat photo — it cannot hide your own clothes underneath, show the back of a
garment or simulate real cloth. `frontend/tryon3d.js` and `avatar3d.js` (the old 3D experiments) are no longer loaded.

**Phones**
- Browsers only allow the camera on **HTTPS** (or `localhost`). `http://192.168.x.x` will NOT work on a phone.
  Quick test: `cloudflared tunnel --url http://localhost:8000` (or `ngrok http 8000`) and open the https link.
  Production: put a TLS terminator in front (Caddy: `your.domain { reverse_proxy nginx:80 }`, or your cloud LB).
- On phones the app uses the front camera at 480×640 portrait, caps at ~30 fps, skips the blur filter,
  and enlarges touch targets. Pose detection and rendering run on the phone; video is never uploaded.
- Scan tips: prop the phone ~2.5 m away at chest height, plain background, whole body in frame.

## 6. Next steps for scale
S3 for storage · Alembic migrations · Celery/RQ if you want retries and priorities ·
Prometheus metrics · CDN for catalog images · GPU workers for the model.
