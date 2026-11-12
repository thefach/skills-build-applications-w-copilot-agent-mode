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

Keep presentation, application logic, and persistence in their existing tiers. The frontend calls the Express API through the shared client and does not access MongoDB directly. The API owns validation, business rules, and response shapes; Mongoose models own persisted-field constraints and defaults. Keep the exercise unauthenticated.

- Presentation tier decision: Use React routes and the shared API client for each capability. Render explicit loading, empty, and accessible error states, and keep API response handling out of presentation-only components where practical.
- Logic tier decision: Implement the `/api/*` contract in Express routes/services. Validate request shape and referenced IDs before persistence, return deliberate 4xx responses for invalid or unknown resources, and avoid leaking Mongoose cast errors as 500 responses.
- Data tier decision: Define required fields, references, and defaults in Mongoose models. Keep seed data representative and idempotent; preserve unrelated records when seeding.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and positive `minutes`; verify both IDs are valid and refer to existing records. Persist the corresponding `user` and `team` references, calculate positive points in the logic tier, and set `occurredAt` when creating the activity.
- Team management approach: Require a name, retain the submitted name and mascot, and default members to an empty array. Validate member references and expose members consistently in create/list responses.
- Leaderboard approach: Sum persisted activity points per team, sort descending by total, and assign ranks starting at 1. Return the documented `teamName`, `totalPoints`, and `rank` fields.
- Workout suggestions approach: Find the user by email, use recent activity to choose a matching workout focus when available (falling back to the user's fitness goal), and include a concise reason. Return a deliberate 4xx for an unknown email.

## Test plan

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks model constraints/defaults and repeatable, non-destructive seeding. Failures indicate a persistence contract or seed behavior regression.
- API logic: `npm run test:api --prefix octofit-tracker/backend` checks request validation, activity/team persistence, leaderboard totals/ranks, and user-specific workout suggestions. Failures indicate an endpoint, validation, or business-rule mismatch.
- Presentation: `npm test --prefix octofit-tracker/frontend` checks API-backed rendering and the loading, empty, and accessible error states for each route. Failures indicate a frontend integration or user-visible state regression.
- Backend build: `npm run build --prefix octofit-tracker/backend` type-checks the backend with TypeScript. Failures indicate backend type or compile errors.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` type-checks the frontend and creates the Vite production build. Failures indicate frontend type, compile, or bundling errors.

Baseline observed on 2026-11-12: the data suite had 4 failures and 1 pass (the seed test passed); the API suite had 4 failures; and the frontend suite had 15 failures. These are starter-contract failures to address in data, logic, then presentation order—not expected passing results.
