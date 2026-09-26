1| https://github.com/user-attachments/assets/5d5916a7-88dc-4467-b8f3-cbce26f64364
2| 
3| # farm-ts
4| 
5| Minimal split backend/frontend starter: **FastAPI + MongoDB** behind a
6| **Vite + React 19 + TypeScript** frontend, joined by a small typed fetch layer
7| over `/api`. This is a bare skeleton — no app features are implemented. Build on
8| top of it.
9| 
10| ## Layout
11| 
12| ```
13| farm-ts/
14|   backend/   FastAPI + motor (async MongoDB) + Pydantic v2 — python
15|   frontend/  Vite + React 19 + Tailwind v4 + shadcn/ui (TypeScript strict)
16|   tests/     Playwright e2e workspace (pre-scaffolded)
17| ```
18| 
19| ## Running
20| 
21| Two separate processes, managed by supervisor in the pod (see "Pod conventions"
22| below); to run them by hand from two terminals instead:
23| 
24| ```bash
25| cd backend && uvicorn server:app --host 0.0.0.0 --port 8001 --reload   # http://localhost:8001
26| cd frontend && yarn dev                                                # http://localhost:3000
27| ```
28| 
29| ## The `/api` proxy convention
30| 
31| Every backend route lives under `/api` (the backend mounts one
32| `APIRouter(prefix="/api")`), and the frontend dev server
33| (`frontend/vite.config.ts`) proxies `/api/*` to `http://localhost:8001`. So
34| frontend code always calls a **relative** path — `apiGet("/status")` →
35| `/api/status` — and never an absolute backend URL. The same code works in dev
36| (via the Vite proxy) and in production (once both are served behind a single
37| origin).
38| 
39| ## Backend
40| 
41| FastAPI, async throughout. `python` is the app interpreter; backend deps are
42| pip-installed from `backend/requirements.txt`.
43| 
44| - **Entry point**: `backend/server.py` — creates `app = FastAPI()`, creates
45|   `api_router = APIRouter(prefix="/api")`, registers routes **on the router**,
46|   and calls `app.include_router(api_router)` at the bottom. CORS middleware is
47|   added from `CORS_ORIGINS`. Never hang a route directly off `app` — it would
48|   land outside `/api` and the Vite proxy would not reach it.
49| - **The route pattern** (copy `status` in `server.py`):
50|   1. a Pydantic model per request body and per response
51|      (`StatusCheckCreate` / `StatusCheck`);
52|   2. an `async def` handler decorated with
53|      `@api_router.post("/status", response_model=StatusCheck)`;
54|   3. `await` the motor call inside it.
55|   FastAPI validates the request against the Pydantic model before your handler
56|   runs — a malformed body never reaches your code, it gets an automatic `422`
57|   with a `{"detail": [...]}` body.
58| - **Growing the backend**: as `server.py` gets crowded, move models to
59|   `backend/models/` and routers to `backend/routers/` (one module per resource,
60|   each exporting its own `APIRouter`, mounted from `server.py` via
61|   `api_router.include_router(...)` or `app.include_router(...)` with the `/api`
|   prefix preserved).
62| - **MongoDB**: import the shared handle — `from lib.db import client, db`
63|   (`backend/lib/db.py` self-loads environment variables before reading them). Use it from
64|   `server.py`, every router, and standalone scripts like `seed.py`; never
65|   construct another `AsyncIOMotorClient`. Collections are attributes:
66|   `await db.status_checks.insert_one(...)`, `await db.status_checks.find().to_list(1000)`.
67|   Motor connects lazily, so importing `server` never blocks on Mongo. `pymongo`
|   is installed too if you need a sync client in a script.
68| - **Ids**: documents use a string `id` (`uuid4`) field, not Mongo's `ObjectId`
69|   — `ObjectId` is not JSON-serializable and leaks into response bodies. Keep the
70|   `uuid4` default-factory pattern from `StatusCheck`.
71| - **Config**: environment variables such as `MONGO_URL`, `DB_NAME`, and
72|   `CORS_ORIGINS` should be loaded before app startup and read with `os.environ`.
73|   `server.py` loads them with `python-dotenv` above its local imports, and
74|   `lib/db.py` self-loads them so standalone scripts inherit them too. The pod
75|   runs `mongod` locally, so `MONGO_URL` points at `localhost`. Add new
76|   secrets/config here; read them with `os.environ`.
77| - **Dates**: `backend/lib/dates.py` — `today_iso(tz=None)`. The pod clock is
78|   UTC; anchor "today" server-side with this, never with client-side date math.
79| - **Interactive check**: `cd /app/backend && python -c 'import server'` catches
80|   syntax/import errors without waiting for the supervisor log.
81| 
82| ## Frontend
83| 
84| - Vite + React 19 + TypeScript strict, dev server on port `3000`.
85| - Tailwind CSS v4 (via the `@tailwindcss/vite` plugin — no separate
86|   `tailwind.config.js` needed) + shadcn/ui, initialized with the `base-nova`
87|   style and `neutral` base color, `@` path alias (`@/*` → `src/*`) wired in both
88|   `tsconfig.app.json`/`tsconfig.json` and `vite.config.ts`.
89| - `react-router-dom` and `motion` are preinstalled — don't re-add them. `src/App.tsx`
90|   is the `<Routes>` table and nothing else; screens live in `src/pages/*.tsx` and are
91|   imported as `@/pages/<Name>`. `src/pages/Home.tsx` ships as the worked example. Add
92|   a `<Route>` for every page you write, in the same edit that creates the page — a
93|   page with no route is unreachable, and any URL without a matching `<Route>` renders a
94|   **blank page** — `<Routes>` matches nothing and mounts nothing.
95| - Components installed under `src/components/ui/`: button, card, input, label,
96|   select, dialog, sheet, tabs, badge, calendar, sonner, textarea, table, popover,
97|   dropdown-menu, checkbox. Add more with `npx shadcn@latest add <component>`.
98| - `src/lib/api.ts` — the typed fetch layer: `apiGet<T>`, `apiPost<T>`,
99|   `apiPut<T>`, `apiPatch<T>`, `apiDelete<T>`, all relative to base `/api`,
100|   throwing `ApiError` (with `status` and the parsed body) on any non-2xx.
101|   **Nothing infers across the Python boundary** — you declare the response type
102|   yourself as a TS interface mirroring the endpoint's Pydantic model, and keeping
103|   the two in sync is a manual discipline. When you change a Pydantic model,
104|   change its TS interface in the same edit.
105| - `src/pages/Home.tsx` is a minimal example of the wiring: TanStack Query's `useQuery`
106|   with `apiGet<StatusCheck[]>('/status')` as the `queryFn`. It is a **non-blocking
107|   connectivity probe**, not a proof of the round trip — the result is deliberately
108|   discarded so the splash renders identically with no backend. `apiGet<T>` does no
109|   runtime validation either; `T` is your assertion, not a check. See the
110|   static-preview rule in `TEMPLATE.md` §4 for why no page may be gated on a fetch.
111| 
112| ## TypeScript
113| 
114| `frontend/tsconfig.app.json` / `tsconfig.node.json` have `strict: true`. In the
115| pod:
116| 
117| ```bash
118| cd frontend && yarn typecheck
119| ```
120| 
121| — plain `tsc --noEmit` run from `frontend/` checks ZERO files (root tsconfig uses
122| project references with `"files": []`) and exits 0 even with type errors. Always
123| use `-b` for the frontend. Lint with `cd frontend && yarn lint` (oxlint).
124| 
125| ## Data fetching
126| 
127| TanStack Query is wired: `QueryClientProvider` in `src/main.tsx`, `useQuery` demo
128| in `src/pages/Home.tsx` (see above). Use `useQuery`/`useMutation`, not
129| fetch-in-`useEffect`.
130| 
131| ## Completion gate (tier 1)
132| 
133| When the build is complete, run tier 1 once, all in the same turn: a curl smoke
134| over the key `/api` endpoints (assert status AND a response field, plus one
135| negative case), `cd frontend && yarn typecheck`, and ONE happy-path browser pass
136| through the core user journey. Clean on all three → finish; any failure is a real
137| bug — fix it, re-run the failed check, and escalate to the testing subagent.
138| No routine typecheck/lint/smoke passes during the build — tier 1 runs exactly once.
139| 
140| 
141| ## Testing
142| 
143| Two lanes.
144| 
145| **Backend (pytest)** — specs in `backend/tests/` as `test_*.py`, run with:
146| 
147| ```bash
148| cd /app/backend && pytest
149| ```
150| 
151| `backend/pytest.ini` is canonical: `addopts = -n 2 --dist loadscope` (pytest-xdist,
152| already parallel — do not pass your own `-n`) and `asyncio_mode = auto` (so
153| `async def test_...` needs no marker). Serial is `-n 0`, **never**
154| `-p no:xdist` (that errors, because `addopts` still passes `-n`/`--dist`).
155| `backend/tests/conftest.py` is pre-scaffolded — a sync `client` fixture
156| (`httpx.Client` rooted at `/api`), an async `aclient`, and an `api_url()` helper,
157| all pointed at `BACKEND_URL` (default `http://localhost:8001`). Tests hit the
158| live uvicorn process, so the app under test is the one the browser sees. Add
159| app-specific fixtures below the marker; do not re-create the file.
160| 
161| **Frontend (Playwright)** — `/app/tests/` is pre-scaffolded:
162| `playwright.config.ts` (canonical — edit the marked lines only),
163| `fixtures/helpers.ts`, and a `package.json` that resolves
164| `@playwright/test@1.62.0` (node_modules baked into the image). Write specs into
165| `tests/e2e/`. Do NOT re-create the config/helpers or install/upgrade playwright —
166| matching Chromium browsers live at `/pw-browsers`.
167| 
168| The backend lane is pytest: this template's backend is Python, so `vitest` does
169| not apply to it.
170| 
171| ## Pod conventions
172| 
173| This template runs under supervisord in the Emergent agent pod — supersedes any
174| local-run instructions above.
175| 
176| - Backend, frontend, and `mongod` are each a supervisor program. After code or
177|   config changes, restart and wait for readiness:
178| 
179|   ```bash
180|   sudo supervisorctl restart frontend backend
181|   until curl -sf -o /dev/null http://localhost:3000; do sleep 2; done
182|   ```
183| 
184| - Status, only after a restart you triggered:
185|   `sudo supervisorctl status frontend backend`. Logs:
186|   `/var/log/supervisor/backend.err.log`, `backend.out.log`,
187|   `frontend.err.log`.
188| - App in a browser: the pod's preview URL (frontend, port `3000`). Backend API
189|   directly at port `8001`.
190| - `mongod` runs locally in the pod (`--bind_ip_all`); `MONGO_URL` in
191|   `backend/.env` points at `localhost`, no separate Mongo container.
192| - Both dev servers hot-reload on file edits (uvicorn `--reload` for the backend,
193|   Vite HMR for the frontend); no rebuild step needed for normal iteration. A
194|   restart is still needed after changing `.env`, `requirements.txt`, or
195|   `vite.config.ts`.
196| 
