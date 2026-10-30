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

Record your plan here before editing code.

- Presentation tier decision: Keep all route/page components in the React app, use a shared fetch-based client to call `/api`, and render explicit loading, empty, and error states so the UI matches the contract before considering styling polish.
- Logic tier decision: Build the Express API around small route handlers that validate input, delegate to services or direct model queries, and return consistent 4xx responses for invalid IDs or missing users rather than leaking cast errors or 500s.
- Data tier decision: Model the application around MongoDB collections for users, teams, activities, and workouts, with Mongoose validation and seed data that are idempotent and representative of the real school fitness domain.
- Activity tracking approach: Store `user`, `team`, `type`, `minutes`, `points`, and `occurredAt`; validate that the referenced user and team exist; compute points from the submitted minutes when creating activity records.
- Team management approach: Require `name` and `mascot`, default `members` to an empty array, and preserve `members` as a list of user IDs when teams are created and returned.
- Leaderboard approach: Aggregate activity points by team, sort descending by total points, and assign a 1-based rank to each team entry in the leaderboard response.
- Workout suggestions approach: Match the current user by email, inspect recent activity patterns such as the user’s goal or activity type, and return a ranked list of workout suggestions whose `focus` and `reason` align with that pattern.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`  
  Failing result: the schema tests show that `User`, `Team`, `Activity`, and `Workout` are missing required validation. This is the first problem to fix because it blocks the data contract and strengthens the database model.
- API command: `npm run test:api --prefix octofit-tracker/backend`  
  Failing result: invalid API requests return `501` instead of a controlled `4xx`, the leaderboard is empty, and workout suggestions do not match the required user pattern. This shows the logic tier is still stubbed or incomplete.
- Frontend command: `npm test --prefix octofit-tracker/frontend`  
  Failing result: the UI still renders “Implementation pending,” and the app does not yet show loading, empty, or accessible error states. This confirms the presentation layer is not yet consuming the API contract correctly.
- Backend build command: `npm run build --prefix octofit-tracker/backend`  
  Expected after the data and API fixes: TypeScript should compile cleanly once the route and model contracts are implemented.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`  
  Expected after the API and UI contract are aligned: the React app should type-check and build successfully with the finalized route and fetch behavior.

These commands should be run in order: data contract, API contract, then frontend tests, followed by both build checks. This sequence minimizes rework because the data layer establishes validation, the logic layer enforces behavior, and the presentation layer consumes the final contract last.
