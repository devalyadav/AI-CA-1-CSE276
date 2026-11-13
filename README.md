# PantryChef

PantryChef is a responsive Progressive Web App (PWA) that generates personalized cooking recipes from the ingredients a user already has on hand. It integrates with Google Gemini AI to propose practical, safe meals that minimize food waste while adhering to dietary restrictions, cuisine preferences, time constraints, and serving sizes.

The application is structured as a full-stack TypeScript monorepo consisting of an Express.js backend and a React (Vite) frontend.

---

## Architecture Overview

```
+-------------------------------------------------------------+
| Browser / PWA Client                                        |
| - React 18, TypeScript, Tailwind CSS, Vite                  |
| - Service Worker & Web Manifest (vite-plugin-pwa)            |
| - Local persistence: IndexedDB (idb-keyval) + localStorage  |
| - Never accesses Gemini API or GEMINI_API_KEY directly     |
+------------------------------+------------------------------+
                               |
                   Fetch API   | (proxied to :3001 in dev)
                               v
+-------------------------------------------------------------+
| Node.js Server (Port 3001)                                  |
| - Express.js, TypeScript, Helmet, CORS, express-rate-limit  |
| - Shared Zod validation schemas (@pantrychef/shared)        |
| - RecipeGenerator provider abstraction                      |
|   ├── GeminiRecipeGenerator (@google/genai SDK)             |
|   └── MockRecipeGenerator (USE_MOCK_GEMINI=true fallback)   |
| - Serves static client bundle (client/dist) in production   |
+------------------------------+------------------------------+
                               |
              @google/genai    | Secure server-to-server calls
                               v
+-------------------------------------------------------------+
| Google Gemini API (gemini-3.8-flash)                        |
| - Enforces structured JSON output schema                    |
| - Strict culinary prompt isolation and safety rules         |
+-------------------------------------------------------------+
```

### Security & Key Isolation
- The Gemini API key (`GEMINI_API_KEY`) is stored strictly in server environment variables.
- It is never exposed in browser bundles, client HTML, network responses, logs, or error payloads.
- An automated secrets audit script (`npm run check:secrets`) scans source code and build artifacts to prevent accidental key leakage or frontend references to `GEMINI_API_KEY`.
- All user ingredients are treated strictly as raw text data within isolated system instructions, neutralizing potential prompt injection attempts.
- In production, CORS is restricted to the configured application origin, and inbound payloads are capped at 10 KB with rate limiting applied to recipe generation endpoints.

---

## Project Structure

```
pantrychef/
├── client/                     # Frontend React application
│   ├── public/                 # Static assets, SVG and PNG icons, manifest
│   ├── src/
│   │   ├── api/                # Typed API client for backend communication
│   │   ├── components/         # Accessible UI components (Chips, Filters, Cards)
│   │   ├── context/            # React Context state (Favorites, Shopping List)
│   │   ├── pages/              # Route pages (Home, Favorites, Shopping List)
│   │   ├── storage/            # Versioned IndexedDB and localStorage resilience
│   │   └── tests/              # Frontend unit and component tests
│   ├── index.html              # HTML shell
│   ├── tailwind.config.js      # Design tokens (colors, typography, tap targets)
│   └── vite.config.ts          # Vite build and PWA configuration
├── server/                     # Backend Node.js Express server
│   ├── src/
│   │   ├── middleware/         # Error handler, rate limiter, request validator
│   │   ├── routes/             # Health and recipe endpoints
│   │   ├── services/           # Gemini and Mock recipe generators
│   │   ├── utils/              # Sanitization and custom error classes
│   │   ├── app.ts              # Express application configuration
│   │   ├── config.ts           # Environment configuration loader
│   │   └── index.ts            # Entrypoint listening on PORT
│   └── tests/                  # Backend validation, error mapping, and E2E tests
├── shared/                     # Shared TypeScript schemas and contracts
│   └── src/
│       ├── index.ts
│       └── schema.ts           # Zod schemas for Recipe, Requests, and Errors
├── scripts/
│   ├── check-secrets.mjs       # Secret scanning verification script
│   ├── generate-icons.mjs      # SVG-to-PNG PWA icon generator
│   └── verify-e2e.mjs          # End-to-end integration verification script
├── .env.example                # Sample environment variables
├── .gitignore                  # Git ignore rules
├── package.json                # Root monorepo workspace configuration
├── tsconfig.base.json          # Shared TypeScript compiler options
└── README.md                   # Documentation
```

---

## API Contract

All endpoints exchange JSON. Any failure produces a consistent error shape:
```json
{
  "error": {
    "code": "STRING_CODE",
    "message": "User-friendly description"
  }
}
```

Standard error codes:
- `GEMINI_KEY_MISSING` (HTTP 503): Returned when the server does not have `GEMINI_API_KEY` set and mock mode is off.
- `INVALID_REQUEST` (HTTP 400): Request failed Zod validation (e.g., >30 ingredients, ingredient length >40 chars, missing required fields).
- `RATE_LIMITED` (HTTP 429): Client exceeded request frequency thresholds.
- `GEMINI_UNAVAILABLE` (HTTP 502): The Gemini API service is unreachable or returned an upstream error.
- `GEMINI_BAD_RESPONSE` (HTTP 502): Gemini produced an output that failed schema validation after a retry.
- `TIMEOUT` (HTTP 504): The generation request exceeded the server's timeout window.

### Endpoints

#### 1. Health Check
`GET /api/health`
- **Response** (HTTP 200):
  ```json
  {
    "ok": true,
    "geminiConfigured": false
  }
  ```
  Note: Never returns the key value or any part of it.

#### 2. Generate Recipes
`POST /api/recipes`
- **Request Body**:
  ```json
  {
    "ingredients": ["pasta", "tomatoes", "garlic", "olive oil"],
    "diets": ["Vegetarian"],
    "cuisine": "Italian",
    "maxMinutes": 30,
    "servings": 2,
    "count": 3,
    "exclude": []
  }
  ```
- **Response** (HTTP 200):
  ```json
  {
    "recipes": [
      {
        "id": "c1f7a0b3-90d1-4e4b-81d5-2e6b2f4c9c10",
        "title": "Rustic Tomato Garlic Pasta",
        "summary": "A fragrant, comforting skillet pasta tossing pantry tomatoes and garlic in warm olive oil.",
        "timeMinutes": 20,
        "servings": 2,
        "ingredients": [
          { "name": "pasta", "quantity": "200g", "haveIt": true },
          { "name": "tomatoes", "quantity": "2 whole, diced", "haveIt": true },
          { "name": "garlic", "quantity": "3 cloves, minced", "haveIt": true },
          { "name": "olive oil", "quantity": "2 tbsp", "haveIt": true },
          { "name": "fresh basil", "quantity": "1 handful", "haveIt": false }
        ],
        "steps": [
          "1. Bring a pot of salted water to a rolling boil and cook pasta until al dente.",
          "2. Heat olive oil in a wide pan over medium heat and sauté minced garlic until fragrant (60 seconds).",
          "3. Add diced tomatoes and simmer for 6 minutes until broken down into a rustic sauce.",
          "4. Toss cooked pasta into the skillet, coating strands thoroughly in the sauce.",
          "5. Garnish with fresh basil leaves and serve immediately."
        ]
      }
    ]
  }
  ```

#### 3. Regenerate Single Recipe
`POST /api/recipes/regenerate`
- **Request Body**: Same fields as `/api/recipes`, plus `replaceTitle` (the title being replaced) and `exclude` (titles to not repeat).
  ```json
  {
    "ingredients": ["pasta", "tomatoes", "garlic"],
    "diets": ["Vegetarian"],
    "cuisine": "Italian",
    "maxMinutes": 30,
    "servings": 2,
    "count": 3,
    "replaceTitle": "Rustic Tomato Garlic Pasta",
    "exclude": ["Rustic Tomato Garlic Pasta"]
  }
  ```
- **Response** (HTTP 200):
  ```json
  {
    "recipe": {
      "id": "e4f8a1c2-55d2-4b3a-92a1-1e5c3b4a8e21",
      "title": "Crispy Garlic Tomato Bake",
      "summary": "An alternative oven-baked variation roasted at high heat.",
      "timeMinutes": 25,
      "servings": 2,
      "ingredients": [ ... ],
      "steps": [ ... ]
    }
  }
  ```

---

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `GEMINI_API_KEY` | Google Gemini API key obtained from Google AI Studio. | `""` (Unset) |
| `GEMINI_MODEL` | Fast text Gemini model identifier. | `gemini-3.8-flash` |
| `PORT` | Local port for Express backend server. | `3001` |
| `USE_MOCK_GEMINI` | When set to `true`, uses MockRecipeGenerator for tests or demonstrations without consuming API quota. | `false` |
| `CLIENT_ORIGIN` | Allowed client origin for CORS in production. | `http://localhost:5173` |

---

## Local Setup and Running

### Prerequisites
- Node.js 20 or higher
- npm 10 or higher

### 1. Install Dependencies
```bash
npm install
```

### 2. Configure Environment Variables
Copy `.env.example` to `.env`:
```bash
cp .env.example .env
```
Open `.env` and set your key:
```env
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-3.8-flash
PORT=3001
USE_MOCK_GEMINI=false
```
*Note: If you do not have an API key immediately available, set `USE_MOCK_GEMINI=true` to test the entire application end-to-end with realistic mock data.*

### 3. Development Mode
Runs both the Express server on port 3001 and the Vite development server on port 5173 concurrently. Vite proxies `/api` calls directly to port 3001.
```bash
npm run dev
```
Open `http://localhost:5173` in your browser.

### 4. Production Build & Execution
Builds the shared types, client PWA bundle, and backend TypeScript sources. Then starts the unified Node server which serves both the frontend SPA and the backend API from a single process:
```bash
npm run build
npm start
```
Open `http://localhost:3001` in your browser.

---

## Progressive Web App (PWA) Features

PantryChef is built as a standalone-capable PWA using `vite-plugin-pwa`:
- **Web App Manifest**: Configured with theme color (`#CE3E22`), cream background (`#FAF7F2`), standalone display mode, and responsive icons.
- **App Icons**: 192x192 PNG, 512x512 PNG, and 512x512 maskable PNG icons generated from master vector designs using `sharp`.
- **Offline Capabilities**:
  - App shell, scripts, CSS, and SVG icons are precached by the service worker.
  - The **Favorites** page and **Shopping List** are persisted locally via a resilient storage layer combining IndexedDB (`idb-keyval`) and `localStorage` fallback. Both pages work offline without network connectivity.
  - When offline, attempting to generate recipes displays an informative banner informing the user that an active connection is required to contact the recipe service.
- **Install Flow**:
  - Automatically captures the browser's `beforeinstallprompt` event and surfaces an "Install App" button in navigation.
  - Detects iOS Safari users and renders clear step-by-step instructions (Tap Share, then select "Add to Home Screen").

---

## Testing & Verification Suite

PantryChef includes unit and integration tests across both workspaces.

### Run All Tests
```bash
npm test
```
Executes:
- **Server Tests**: 15 tests covering input validation (character caps, serving bounds, 10 KB body limit, malformed JSON), error mapping across all 6 API error codes, and unique title generation on recipe regeneration.
- **Client Tests**: 16 tests covering chip keyboard interactions (Enter, comma, Backspace), deduplication, max limits, storage corruption recovery, shopping list duplicate merging, and error banner interactions.

### Run Type Checking & Linting
```bash
npm run lint
```

### Run Secret Audit
```bash
npm run check:secrets
```
Verifies that no Google API keys ("AIza..." prefix) or client references to `GEMINI_API_KEY` exist in the codebase or build artifacts.

---

## Deployment to a Single Node Host

PantryChef is designed to run on any single container or Node host (such as Render, Railway, Fly.io, Heroku, or Google Cloud Run).

### Build & Start Workflow on Host
1. **Build command**:
   ```bash
   npm install && npm run build
   ```
2. **Start command**:
   ```bash
   npm start
   ```
3. **Environment Variables**:
   Configure these platform secrets in your hosting dashboard:
   - `GEMINI_API_KEY`: Your production Google Gemini API key.
   - `GEMINI_MODEL`: `gemini-3.8-flash`
   - `NODE_ENV`: `production`
   - `PORT`: Provided automatically by the host (defaults to `3001` if unset).
   - `CLIENT_ORIGIN`: Your custom domain (e.g., `https://pantrychef.yourdomain.com`).

The server automatically detects the built client assets in `client/dist`, serves them with secure HTTP caching headers, handles SPA client-side routing fallbacks, and serves the `/api` endpoints simultaneously.
