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

- Presentation tier decision: Build the React routes as thin data consumers: each page fetches from `/api/*`, renders loading, empty, and API-error states, and keeps UI text/accessible roles aligned with the contract. The route components should not own business logic; they should simply map API payloads into cards, tables, and lists.
- Logic tier decision: Keep the Express app organized by capability and service functions so validation, request parsing, aggregation, and 4xx behavior live in the backend instead of the database layer. The API should convert invalid IDs, missing user/team records, and missing names into deliberate 4xx responses rather than unhandled cast errors.
- Data tier decision: Treat the schemas as the source of truth for required properties, defaults, and relationships. `User`, `Team`, `Activity`, and `Workout` should all validate required fields at the model layer; the seed script should insert idempotent fixtures without mutating unrelated data.
- Activity tracking approach: Validate `userId` and `teamId` against existing records, reject malformed or missing references with a non-500 error, then add `points` and `occurredAt` on creation. `GET /api/activities` should return stored activities in a stable list, and the created resource should match the submitted values plus computed metadata.
- Team management approach: Require a team `name` with `400` on invalid payloads, keep `members` as an array defaulting to `[]`, and preserve the submitted `mascot` while returning member IDs in list responses. The API should represent team membership in a way the frontend can render without extra transformation.
- Leaderboard approach: Aggregate points per team across activities and rank entries with a 1-based `rank`, sorted highest total first. The calculation belongs in the backend so the frontend only needs to display the ranking payload.
- Workout suggestions approach: Match `:userEmail` to a seeded user, infer the recommendation from recent activity patterns or the user's `fitnessGoal`, and return the top match with a clear `reason` string. If no user exists, respond with a deliberate 4xx (`404` is acceptable) rather than a generic internal error.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
  - Current failure meaning: the model validation is incomplete; `User`, `Team`, `Activity`, and `Workout` are not enforcing required fields, so the schema-level tests fail before any routes run.
- API command: `npm run test:api --prefix octofit-tracker/backend`
  - Current failure meaning: the route/service layer is still stubbed or missing; invalid requests return `501` instead of `400`, leaderboard aggregation is empty, and workout suggestions do not match the user pattern or reason contract.
- Frontend command: `npm test --prefix octofit-tracker/frontend`
  - Current failure meaning: the presentation layer is still using placeholder content (`Implementation pending`) and never enters the loading/empty/error UI states the contract expects.
- Backend build command: `npm run build --prefix octofit-tracker/backend`
  - Use this after the model and API contract pass to confirm the TypeScript server compiles cleanly.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
  - Use this after the UI contract passes to confirm the React app builds without type or routing issues.

The implementation order is intentionally: data -> logic -> presentation. This matches the dependency chain in the app: the API relies on valid Mongoose models, and the frontend relies on the API contract rather than working around backend gaps.
