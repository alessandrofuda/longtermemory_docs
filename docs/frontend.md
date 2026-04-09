# Frontend — React SPA

Standalone React Single Page Application powering [longtermemory.com](https://longtermemory.com). It communicates with the Laravel backend exclusively via REST API. Frontend and backend are fully decoupled and can be developed and deployed independently.

## Tech Stack

| Tool | Version | Purpose |
|------|---------|---------|
| React | 19 | UI framework |
| TypeScript | latest | Type safety |
| Vite | 7 | Build tool and dev server |
| Tailwind CSS | 4 | Utility-first styling |
| shadcn/ui (New York) | — | Component library |
| Axios | — | HTTP client with interceptors |
| React Router | — | Client-side routing |
| Stripe.js + @stripe/react-stripe-js | — | Payment processing |
| React Helmet Async | — | SEO meta tag management |
| Puppeteer | — | Build-time prerendering for SEO |

---

## Development Commands

```bash
cd frontend
npm install           # Install dependencies

npm run dev          # Dev server → http://localhost:5173
npm run build        # Production build → dist/
npm run build:prerender  # Build + prerender landing page for SEO
npm run lint         # ESLint
npm run types        # TypeScript type check
npm run preview      # Preview production build locally
```

---

## Environment Variables

Create `.env.development` (for local dev) and `.env.production` (for production builds):

```env
VITE_API_BASE_URL=http://localhost:8080   # Backend API base URL
VITE_ENV=development
VITE_STRIPE_PUBLIC_KEY=pk_test_...        # Stripe publishable key
```

---

## Project Structure

```
src/
├── api/              # API client modules (one per feature)
│   ├── auth.ts       # Authentication (magic link, exchange, logout, user, timezone)
│   ├── projects.ts   # Project CRUD
│   ├── sources.ts    # Document/weblink upload and URL check
│   └── study-plans.ts  # Study plan generation, polling, sessions, evaluation
│
├── types/            # TypeScript interfaces (one file per feature)
│   ├── auth.ts       # User, Plan types
│   ├── projects.ts
│   ├── sources.ts    # StoredDocument, WebLinkWithMetadata, CheckUrlResponse
│   └── study-plans.ts  # StudyPlanItem, StudySession, EvaluateQAItemPayload
│
├── components/
│   ├── ui/           # shadcn/ui components + shared UI (LoadingSpinner)
│   ├── dashboard/    # Dashboard-specific components
│   │   ├── TodayStatistics.tsx  # Study session stats with progress bar
│   │   ├── QAItemDisplay.tsx    # Q&A card with answer toggle + evaluation
│   │   └── WeblinksSection.tsx
│   └── *.tsx         # Landing page sections (HeroSection, PricingSection, etc.)
│
├── pages/            # Route-level components
│   ├── Landing.tsx        # Authenticated landing (upload + pricing)
│   ├── LandingPublic.tsx  # Static landing (SEO target for non-auth users)
│   ├── Login.tsx
│   ├── AuthCallback.tsx   # OAuth callback handler
│   ├── Dashboard.tsx
│   ├── StudyPlan.tsx      # View generated Q&A pairs
│   ├── StudySession.tsx   # Interactive spaced repetition session
│   └── Profile.tsx
│
├── hooks/            # Custom hooks (useLogout, etc.)
├── contexts/         # UserContext
├── lib/              # Utilities (cn() for Tailwind, getErrorMessage())
├── scripts/          # Build scripts (prerender.mjs)
├── App.tsx           # Root component with routing and providers
└── main.tsx          # Entry point with HelmetProvider + Elements provider
```

### Path Aliases

Use `@/*` for all internal imports:

```typescript
import { Button } from '@/components/ui/button'
import { authApi } from '@/api/auth'
import type { User } from '@/types/auth'
```

---

## Routing

| Route | Component | Access |
|-------|-----------|--------|
| `/` | `LandingRoute` → `Landing` or `LandingPublic` | Public |
| `/login` | `Login` | Public |
| `/auth/callback` | `AuthCallback` | Public |
| `/dashboard` | `Dashboard` | Auth required |
| `/study-plan/pr/:projectId` | `StudyPlan` | Auth required |
| `/study-session/pr/:projectId` | `StudySession` | Auth required |
| `/profile` | `Profile` | Auth required |

Protected routes check for `auth_token` in `localStorage` and redirect to `/login` if missing.

**Landing page routing**: The `/` route renders `LandingPublic` (static, SEO-optimized) for unauthenticated users and `Landing` (full app with upload forms) for authenticated users:

```tsx
function LandingRoute() {
  const isAuthenticated = !!localStorage.getItem('auth_token');
  return isAuthenticated ? <Landing /> : <LandingPublic />;
}
```

---

## Authentication

LongTerMemory uses **magic link** authentication (no passwords).

### Flow

1. User enters their email on `/login`
2. Backend sends a magic link via `POST /api/auth/magic-link`
3. User clicks the link — it contains a one-time authorization code
4. `/auth/callback` receives the code and exchanges it for a JWT token via `POST /api/auth/exchange`
5. Token is stored in `localStorage` as `auth_token`
6. All subsequent API requests include `Authorization: Bearer {token}` (managed by Axios interceptors)
7. On successful login, the browser timezone is auto-detected and sent to the backend via `POST /api/user/update-timezone`

---

## API Integration Pattern

1. Define types in `src/types/{feature}.ts`
2. Create an API module in `src/api/{feature}.ts` using the shared Axios instance
3. Import and call in components: `featureApi.method(payload)`

```typescript
// src/api/sources.ts
export const sourcesApi = {
  uploadDocuments: (formData: FormData) =>
    api.post<UploadResponse>('/api/documents', formData),

  checkUrl: (url: string) =>
    api.post<CheckUrlResponse>('/api/check-url', { url }),
};
```

---

## State Management

- **`UserContext`** (`src/contexts/UserContext.tsx`) is the single source of truth for the authenticated user object and plan info.
- Access it with the `useUser()` hook — never fetch `/api/user` independently inside page components.
- Call `refetchUser()` only when the user's data actually changes (e.g., after a plan upgrade or trial start).

```typescript
// Good
const { user, refetchUser } = useUser();

// Bad — duplicates state
const [user, setUser] = useState<User | null>(null);
```

- **`useLogout()`** hook encapsulates logout logic (token removal + navigation). Import it instead of repeating the implementation in every page.

---

## Key Patterns

### StrictMode ref guard

React 19 runs effects twice in development (`<StrictMode>`). For one-time operations (OAuth callback, payment confirmation), use a ref:

```typescript
const hasExecuted = useRef(false);

useEffect(() => {
  if (hasExecuted.current) return;
  hasExecuted.current = true;
  // Critical operation here
}, []);
```

### Polling for async job progress

Study plan generation is async (Celery task). The frontend polls for status every 5 seconds:

```typescript
useEffect(() => {
  if (!pollingJobId) return;

  const interval = setInterval(async () => {
    const status = await studyPlansApi.checkStatus(pollingJobId);
    setStudyPlanProgress(status.progress ?? 0);

    if (status.ready) {
      setPollingJobId(null); // stops polling
    }
  }, 5000);

  return () => clearInterval(interval); // cleanup on unmount
}, [pollingJobId]);
```

### Spaced repetition self-evaluation

After viewing an answer, the user self-assesses with one of four difficulty levels. The evaluation is saved to the API and the next question is fetched automatically:

| Level | Interval | Color |
|-------|----------|-------|
| Again | 1 min | Red |
| Hard | 10 min | Yellow |
| Good | 4 days | Blue |
| Easy | 8 days | Green |

### Error handling

Use `getErrorMessage()` from `@/lib/utils` for consistent error extraction across the app:

```typescript
import { getErrorMessage } from '@/lib/utils';

try {
  await api.post('/endpoint', data);
} catch (error) {
  setError(getErrorMessage(error, 'Something went wrong'));
}
```

---

## SEO

The landing page uses a two-pronged SEO approach:

- **Runtime**: `LandingPublic.tsx` renders a fully static, no-API version of the landing page for unauthenticated visitors and search engine crawlers.
- **Build time**: `npm run build:prerender` runs a Puppeteer script (`scripts/prerender.mjs`) that renders `LandingPublic` in headless Chrome and saves the complete HTML to `dist/index.html`.

Dynamic meta tags (Open Graph, Twitter Card, canonical URL) are managed via `react-helmet-async` in each page component.

---

## Reusable Components

### `LoadingSpinner` (`src/components/ui/LoadingSpinner.tsx`)

```tsx
<LoadingSpinner size="md" color="green" text="Loading..." />
```

Props: `size` (`sm` | `md` | `lg`), `color` (`green` | `blue` | `white`), `text` (optional).

### `TodayStatistics` (`src/components/dashboard/TodayStatistics.tsx`)

Displays a 4-column grid of study session stats (questions to review, new questions, total, estimated time) with a color-coded stacked progress bar.

### `QAItemDisplay` (`src/components/dashboard/QAItemDisplay.tsx`)

Self-contained Q&A card. Manages answer visibility, displays metadata badges (difficulty, key concepts, scheduled date), shows the spaced repetition evaluation buttons when the answer is visible, and tracks session progress via a progress bar.
