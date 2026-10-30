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

- Presentation tier decision: Keep React components responsible for rendering and user interaction. Use the shared API client with relative `/api` paths, and expose accessible loading, success, empty, and error states for each capability. Do not put persistence or business rules in the browser.
- Logic tier decision: Keep Express route handlers thin: parse requests, call service functions, and translate deliberate service errors into HTTP responses. Put validation and capability rules in the service layer; do not let routes access MongoDB models directly.
- Data tier decision: Use Mongoose schemas to define required fields, references, defaults, and persistence. Keep seed data idempotent and preserve unrelated existing records. Do not add authentication in this exercise.
- Activity tracking approach: Validate the request shape, positive minutes, and well-formed IDs, then confirm the referenced user and team exist before creating an activity. Store the references, type, minutes, computed points, and occurrence time; return deliberate 4xx responses for invalid or missing references.
- Team management approach: Require a non-empty name and mascot, default members to an empty array, and retain the submitted name and mascot when creating and listing teams. Validate member references before associating them.
- Leaderboard approach: Sum activity points by team, order totals from highest to lowest, and return each team name, total points, and a 1-based rank. Keep the aggregation in the service/data access path, not in presentation components.
- Workout suggestions approach: Find the user by decoded email, use their fitness goal and recent activity to prioritize matching workouts, and return an explanatory reason with each suggestion. Return a deliberate 4xx response if no user matches.

## Test plan

- Data tier: `npm run test:data --prefix octofit-tracker/backend` checks model validation and idempotent seeding. A failure in a required-field assertion points to a schema gap; a seed failure points to fixture creation, linking, preservation, or repeatability.
- Logic/API tier: `npm run test:api --prefix octofit-tracker/backend` checks request validation, persistence, leaderboard ordering, and personalized workout suggestions. A `5xx` for invalid input indicates missing deliberate validation; wrong response data indicates a service rule or query problem.
- Frontend: `npm test --prefix octofit-tracker/frontend` checks API requests and accessible loading, success, empty, and error states. A failure indicates a presentation/API contract mismatch.
- Backend build: `npm run build --prefix octofit-tracker/backend` type-checks the Express, service, and model code; a failure indicates a TypeScript contract or implementation error.
- Frontend build: `npm run build --prefix octofit-tracker/frontend` type-checks and bundles the Vite app; a failure indicates a frontend type or production-build issue.
- Exercise validation: `node --test tests/exercise-validation.test.mjs` checks repository and learner-grading contracts. A failure identifies an exercise metadata, workflow-safety, or validation-rule mismatch.

Latest checks on `build-octofit-app`: the data suite has 4 failing model-validation tests and 1 passing idempotent-seed test; the API suite has 4 failures (activity/team creation return `501`, leaderboard is empty, and workout suggestions do not match the user's goal/activity). The frontend suite passes all 15 tests, and both frontend and backend builds pass. The exercise validator passes 36 of 40 tests; all four failures currently report the README Copy Exercise badge owner/name mismatch, which also prevents three fixture checks from reaching their intended assertion.
