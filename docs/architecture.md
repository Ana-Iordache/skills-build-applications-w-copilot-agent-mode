# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier: Keep route-level components focused on rendering and user interaction; use the shared API client for relative `/api` requests and show loading, empty, and accessible error states.
- Logic tier: Keep HTTP validation and response shaping in Express routes, with capability behavior in the existing service layer. Return deliberate 4xx responses for invalid or unknown references instead of exposing database cast errors.
- Data tier: Use the existing Mongoose models as the source of required fields and defaults, and keep representative, idempotent fixtures in the seed script without deleting unrelated records.
- Activity tracking: Validate required input, ID shape, and that both referenced records exist before creating an activity. Compute points and occurrence time on the server; list stored activities through the API.
- Team management: Require a name, preserve submitted name and mascot, and maintain member references through the team model and API.
- Leaderboard: Aggregate activity points by team in the logic tier, sort totals descending, and assign ranks starting at 1.
- Workout suggestions: Find the user by email, use their goal and recent activities to choose a matching workout, and include a concise reason. Return a deliberate 4xx when no user matches.

## Test plan

- Data tier: `npm run test:data --prefix octofit-tracker/backend` (run; 4 failures, 1 pass). Failures indicate missing required model fields or defaults; the seed/idempotency test passes.
- API: `npm run test:api --prefix octofit-tracker/backend` (run; 4 failures). Failures indicate incomplete route/service behavior: activity and team validation return `501`, leaderboard results are empty, and workout suggestions omit the expected reason.
- Presentation: `npm test --prefix octofit-tracker/frontend` (run; 15 failures). Failures indicate the views still report implementation pending or lack API-driven content and loading, empty, and error states.
- Backend build: `npm run build --prefix octofit-tracker/backend` (not run). Use to detect backend TypeScript errors after implementation.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` (not run). Use to detect frontend TypeScript or Vite build errors after implementation.
