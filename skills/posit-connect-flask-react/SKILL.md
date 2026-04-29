---
name: posit-connect-flask-react
description: Details on how to successfully deploy a flask app to posit connect
---
# Skill: Deploy a Flask + React SPA to Posit Connect

## Overview

This skill describes the exact file structure, configuration, and patterns required to successfully deploy a Python Flask backend + React (Vite) frontend as a single application on Posit Connect. Posit Connect mounts apps under dynamic sub-paths (e.g., `/content/<UUID>/`), which breaks naive SPA routing and API calls. This skill documents the proven pattern that handles this correctly.

---

## Critical Concepts

### Posit Connect's Dynamic Base Path

Posit Connect serves every app under a unique, unpredictable sub-path. The app does NOT live at `/`. It lives at something like `/content/<guid>/`. This means:

1. React Router's `basename` must be set dynamically at runtime
2. All `fetch()` calls to the API must be prefixed with the base path
3. All `<script>` and `<link>` asset references in `index.html` must resolve to the correct absolute path
4. Flask's `request.script_root` provides the mount prefix at runtime

### Gunicorn + Fork Workers

Posit Connect runs Flask apps via gunicorn with fork-based workers. Any module-level code (like database connections) runs in the master process BEFORE workers are forked. SQLAlchemy's `Session` class handles this correctly if you return a plain `Session` from your session factory — do NOT wrap it in `@contextmanager`.

---

## Required File Structure

```
project-root/
├── main.py                      # Flask app factory (entrypoint)
├── requirements.txt             # Compiled deps (uv pip compile)
├── pyproject.toml
├── src/
│   ├── __init__.py             # Empty file
│   ├── database.py             # Module-level DB connection + get_session()
│   ├── model.py                # SQLModel table definitions
│   ├── routes.py               # Flask Blueprints (api + ui)
│   ├── static/                 # ← Vite build output lands here
│   │   ├── index.html
│   │   └── assets/
│   │       ├── app.js          # Predictable entry filename
│   │       └── *.css, *.js     # Hashed chunks/assets
│   └── frontend/               # React source (optional to deploy)
│       ├── index.html
│       ├── package.json
│       ├── vite.config.js
│       └── src/
│           ├── main.jsx
│           ├── App.jsx
│           └── lib/
│               └── api.js      # Centralized fetch client
└── .posit/
    └── publish/
        └── <app-name>.toml     # Posit Publisher config
```

---

## File-by-File Implementation

### `main.py`

Minimal app factory. Posit Connect (via gunicorn) looks for a module-level `app` object.

```python
from flask import Flask

from src.routes import api, ui


def create_app() -> Flask:
    app = Flask(__name__, static_folder=None)
    app.register_blueprint(api)
    app.register_blueprint(ui)
    return app


app = create_app()

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

**Key details:**
- `static_folder=None` — disables Flask's built-in static handler so it doesn't intercept requests before the catch-all SPA route
- `api` blueprint registered FIRST (specific `/api/*` routes take priority)
- `ui` blueprint registered SECOND (catch-all `/<path:path>` serves the SPA)

---

### `src/routes.py`

Two Blueprints: `api` for JSON endpoints, `ui` for serving the SPA with base-url injection.

```python
from __future__ import annotations

import os

from flask import Blueprint, jsonify, make_response, request, send_from_directory
from sqlmodel import select

from src.database import get_session
from src.model import MyModel

STATIC_DIR = os.path.join(os.path.dirname(__file__), "static")

api = Blueprint("api", __name__)
ui = Blueprint("ui", __name__)


# ── API routes ────────────────────────────────────────────────────────────────

@api.get("/api/items")
def list_items():
    with get_session() as session:
        items = session.exec(select(MyModel)).all()
        return jsonify([i.model_dump() for i in items])


# ── UI catch-all — serve the React SPA ────────────────────────────────────────

@ui.get("/")
@ui.get("/<path:path>")
def serve_spa(path: str = ""):
    # Serve real static files (JS, CSS, fonts) if they exist on disk
    if path and os.path.exists(os.path.join(STATIC_DIR, path)):
        return send_from_directory(STATIC_DIR, path)

    # For all other paths (SPA routes), serve index.html with base-url injected
    # request.script_root is set by Posit Connect's WSGI middleware to the
    # mount prefix (e.g. "/content/7e8e4d1c-.../")
    base_url = (request.script_root or "").rstrip("/") + "/"

    index_path = os.path.join(STATIC_DIR, "index.html")
    with open(index_path, "r", encoding="utf-8") as f:
        html = f.read()

    # Rewrite the meta tag so React Router and the API client know the base
    html = html.replace(
        '<meta name="base-url" content="/" />',
        f'<meta name="base-url" content="{base_url}" />',
    )

    # Rewrite relative asset paths to absolute ones rooted at the base URL.
    # Without this, a browser at /feedback/RSK-001 would resolve ./assets/app.js
    # to /feedback/assets/app.js instead of /content/<guid>/assets/app.js
    html = html.replace('src="./assets/', f'src="{base_url}assets/')
    html = html.replace('href="./assets/', f'href="{base_url}assets/')

    response = make_response(html)
    response.headers["Content-Type"] = "text/html; charset=utf-8"
    return response
```

**Critical details:**
- `STATIC_DIR` is relative to `__file__` (i.e., `src/static/`)
- The `base-url` meta tag substitution is the core mechanism that makes the SPA work on Connect
- Asset path rewriting (`./assets/` → `{base_url}assets/`) prevents broken references on deep SPA routes
- API routes MUST be registered before the UI catch-all

---

### `frontend/vite.config.js`

```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  // Relative base so asset references in index.html are "./assets/..."
  // Flask's serve_spa rewrites these to absolute paths at runtime
  base: './',
  build: {
    outDir: '../src/static',
    emptyOutDir: true,
    rollupOptions: {
      output: {
        // Predictable entry filename so the src="./assets/app.js" rewrite works
        entryFileNames: 'assets/app.js',
        // Chunks and assets keep hashes for cache-busting
        chunkFileNames: 'assets/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash].[ext]',
      },
    },
  },
  server: {
    proxy: {
      '/api': 'http://localhost:5000',
    },
  },
})
```

**Critical details:**
- `base: './'` — produces relative paths (`./assets/...`) that Flask can rewrite
- `outDir: '../src/static'` — build output goes INSIDE the `src/` package so it's deployed with `/src`
- `entryFileNames: 'assets/app.js'` — predictable name (no hash) so Flask's string replacement in `index.html` works reliably
- Chunk/asset files CAN have hashes — they're loaded at runtime by the browser relative to the page base, which React Router handles correctly via the `basename` prop

---

### `frontend/index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My App</title>
    <!-- Flask rewrites this at serve-time with the Posit Connect mount path -->
    <meta name="base-url" content="/" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="./src/main.jsx"></script>
  </body>
</html>
```

**The `<meta name="base-url" content="/" />` tag is mandatory.** Flask finds it by exact string match and replaces it.

---

### `frontend/src/main.jsx`

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

// Read the runtime base path injected by Flask
const baseMeta = document.querySelector('meta[name="base-url"]')
const basename = baseMeta?.content ?? '/'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App basename={basename} />
  </React.StrictMode>
)
```

---

### `frontend/src/App.jsx`

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom'

export default function App({ basename }) {
  return (
    <BrowserRouter basename={basename}>
      <Routes>
        {/* your routes */}
      </Routes>
    </BrowserRouter>
  )
}
```

**`basename` must be passed to `BrowserRouter`.** This is what makes React Router generate correct links under the Posit Connect sub-path.

---

### `frontend/src/lib/api.js`

```javascript
/**
 * Centralized API client that prefixes every request with the Posit Connect
 * base URL. All components MUST use this instead of raw fetch().
 */

function getBase() {
  const meta = document.querySelector('meta[name="base-url"]')
  const raw = meta?.content ?? '/'
  return raw.endsWith('/') && raw !== '/' ? raw.slice(0, -1) : raw === '/' ? '' : raw
}

async function apiFetch(path, options = {}) {
  const base = getBase()
  const url = `${base}${path}`
  const res = await fetch(url, {
    headers: { 'Content-Type': 'application/json', ...options.headers },
    ...options,
  })
  if (!res.ok) {
    const text = await res.text()
    throw new Error(text || `HTTP ${res.status}`)
  }
  return res.json()
}

export const api = {
  getItems: () => apiFetch('/api/items'),
  getItem: (id) => apiFetch(`/api/items/${encodeURIComponent(id)}`),
  updateItem: (id, data) => apiFetch(`/api/items/${encodeURIComponent(id)}`, {
    method: 'PATCH',
    body: JSON.stringify(data),
  }),
}
```

**NEVER use raw `fetch('/api/...')` in components.** On Posit Connect, `/api/items` resolves to the Connect server root, not the app. All API calls must go through this client which prepends the base path.

---

### Dynamic Content with Root-Relative URLs (Images, Links)

If your React app renders dynamic content (e.g., Markdown from an API) that contains root-relative URLs like `/images/diagram.png`, those URLs will **not** resolve correctly on Posit Connect — the browser will request `https://posit-connect.example.com/images/diagram.png` instead of `https://posit-connect.example.com/content/<guid>/images/diagram.png`.

Flask's `serve_spa` only rewrites the `index.html` asset paths. Dynamic content rendered _after_ page load (fetched via API, rendered by React) must prepend the base path in the React component.

#### Solution: Prepend base path in custom renderers

For `react-markdown` or any component that renders images from dynamic content:

```jsx
import { getBase } from '../lib/api'

// Inside your ReactMarkdown components prop:
components={{
  img: ({ src, alt }) => {
    // Prepend base path to root-relative image URLs so they resolve
    // correctly on Posit Connect (where app is at /content/{guid}/)
    const resolvedSrc = src?.startsWith('/') ? `${getBase()}${src}` : src
    return <img src={resolvedSrc} alt={alt || ''} loading="lazy" />
  },
}}
```

#### Why this is needed:
- Flask rewrites `index.html` at serve-time (before React mounts)
- But images/links inside **dynamically fetched content** (API responses, markdown files) are rendered _after_ mount by React
- React has no built-in mechanism to rebase URLs in rendered content
- The fix must happen at the React component level, using the same `getBase()` utility that API calls use

#### What to prepend:
- `getBase()` returns `""` locally (base-url is `/`, stripped to empty string)
- `getBase()` returns `"/content/<guid>"` on Posit Connect
- So `getBase() + "/images/foo.png"` = `/images/foo.png` locally, `/content/<guid>/images/foo.png` on Connect

#### This applies to:
- `<img>` tags from rendered Markdown
- `<a href="/...">` links in dynamic content pointing to app-internal resources
- Any root-relative URL (`/downloads/file.zip`, `/icons/logo.svg`, etc.) in dynamically rendered HTML

**Rule:** Any React component that renders URLs from dynamic content must resolve root-relative paths through `getBase()`. Only hardcoded `index.html` assets are handled by Flask's rewrite.

---

### `.posit/publish/<app-name>.toml`

```toml
product_type = 'connect'
'$schema' = 'https://cdn.posit.co/publisher/schemas/posit-publishing-schema-v3.json'
type = 'python-flask'
entrypoint = 'main.py'
validate = true
files = [
  '/main.py',
  '/requirements.txt',
  '/.posit/publish/<app-name>.toml',
  '/.posit/publish/deployments/<deployment>.toml',
  '/src',
  '/bin'
]
title = '<app-name>'
secrets = ['SNOWFLAKE_PRIVATE_KEY', 'SNOWFLAKE_USER', 'SNOWFLAKE_ACCOUNT', 'SNOWFLAKE_DATABASE', 'SNOWFLAKE_SCHEMA', 'SNOWFLAKE_ROLE', 'SNOWFLAKE_WAREHOUSE']

[python]
```

**Critical rules:**
- `type = 'python-flask'` — tells Connect to use gunicorn
- `entrypoint = 'main.py'` — gunicorn imports `app` from this file
- `/src` in `files` — recursively includes `src/static/` (the built frontend)
- Do NOT include `/frontend` — raw React source is not needed at runtime and bloats the bundle
- `/bin` includes any local wheel files referenced in `requirements.txt`

---

## Deployment Checklist

1. **Build the frontend:** `cd frontend && npm run build` → produces `src/static/`
2. **Verify `src/static/index.html` exists** and contains `<meta name="base-url" content="/" />`
3. **Verify `src/static/assets/app.js` exists** (predictable filename from Vite config)
4. **Compile requirements:** `uv pip compile pyproject.toml --emit-index-url -o requirements.txt`
5. **Publish via Posit Publisher** — ensure the files list does NOT include `/frontend`

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Gunicorn starts, dies after 2 min (no request logs) | `@contextmanager` on `get_session()` or stale connection pool after fork | Use plain `Session` return (no decorator) |
| App loads but page is blank | React Router `basename` not set | Read `base-url` meta tag and pass to `BrowserRouter` |
| API calls return 404 | Raw `fetch('/api/...')` without base prefix | Use centralized `api.js` client |
| Assets 404 on deep SPA routes (e.g., `/feedback/RSK-001`) | Relative `./assets/` resolves against the SPA route, not the app root | Flask rewrites `./assets/` → `{base_url}assets/` in `serve_spa` |
| Images/media broken (404) in dynamically rendered content | Root-relative URLs (`/images/...`) in markdown/API content don't include the base path | Prepend `getBase()` to root-relative `src`/`href` in React renderers (see "Dynamic Content" section) |
| CSS/JS blocked with MIME type `text/plain` (nosniff) | Posit Connect environment has incomplete system MIME database | Register MIME types explicitly at module level: `mimetypes.add_type("application/javascript", ".js")` etc. |
| `static_folder` conflicts | Flask's built-in static handler intercepts before catch-all | Set `static_folder=None` in `Flask(...)` |
| Bundle includes 200+ chunk files with name collisions | `chunkFileNames: 'assets/[name].js'` (no hash) | Use `'assets/[name]-[hash].js'` for chunks |
| `src/static/index.html` not found at runtime | Frontend not built before deploy, or `outDir` misconfigured | Run `npm run build`, verify `outDir: '../src/static'` |
| Double `create_tables()` in logs | Module imported via two paths (e.g., `src.database` and `database`) | Ensure all imports use `from src.database import ...` (never bare `database`) |

---

## Local Development

```bash
# Terminal 1 — Flask backend
cd project-root
uv run flask --app main run --port 5000 --debug

# Terminal 2 — Vite dev server (with API proxy to Flask)
cd project-root/frontend
npm run dev
```

Vite's dev server proxies `/api` to Flask. The `<meta name="base-url" content="/" />` stays as `/` locally — no rewriting needed.
