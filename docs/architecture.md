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

- **Presentation tier decision**: React components fetch from `/api` endpoints on mount using the global `fetch` API. Each route (Activities, Teams, Leaderboard, Workouts) manages its own loading/error/empty state using `useState` and `useEffect`. State is passed to child display components. Bootstrap classes handle layout and styling.

- **Logic tier decision**: Express routes in `src/routes/api.ts` delegate business logic to service functions in `src/services/octofitService.ts`. Services validate input, check MongoDB references, compute derived fields, and return data or throw 4xx errors (never 500). Error-handling middleware catches service errors and responds with appropriate status codes.

- **Data tier decision**: Mongoose models (`User`, `Team`, `Activity`, `Workout`) enforce schema validation for required fields. The idempotent seed function (`src/scripts/seed.ts`) creates or updates fixtures (Mona, Octocats team, Hubot, Robocats team) without duplicating. Seed preserves existing data not in the fixture set by using `findOneAndUpdate` with upsert or conditional `insertOne`.

- **Activity tracking approach**: `createActivity` validates userId and teamId as valid MongoDB ObjectIds, checks they exist, computes points = minutes, sets occurredAt to now, and saves. Returns 400 for missing/malformed input, 404 for non-existent references.

- **Team management approach**: `createTeam` validates name is present (400 if missing), accepts optional members array, and creates the team. `listTeams` returns all teams with members populated as ObjectId strings.

- **Leaderboard approach**: `getLeaderboard` uses MongoDB aggregation: `$group` by team, `$sum` points, `$sort` descending, `$facet` or loop to add 1-based rank. Returns [{ teamName, totalPoints, rank }].

- **Workout suggestions approach**: `suggestWorkouts` finds the user by email (return empty if not found), fetches their recent activities (last 10), identifies the most common activity type or matches their fitnessGoal, queries workouts with matching focus, and returns the first suggestion with a reason (e.g., "You recently did runs").

## Implementation order (smallest safe steps)

1. **Phase 1 (Data Tier)**: Verify/complete Mongoose models, then implement idempotent seed. Run `npm run test:data` until all pass.
2. **Phase 2 (Logic Tier)**: Implement each service function (activities, teams, leaderboard, suggestions). Run `npm run test:api` until all pass.
3. **Phase 3 (Presentation Tier)**: Add fetch logic and state management to route components. Run `npm test` (frontend) until all pass.
4. **Phase 4 (Validation)**: Build both tiers and manually test end-to-end.

## Test plan

Record the commands you run and what each failure means.

- **Data-tier command**: `npm run test:data --prefix octofit-tracker/backend`
  - **Purpose**: Validate Mongoose models enforce required fields; seed function creates/idempotently updates fixtures.
  - **Expected on first run**: 4–5 passes. All models should enforce their validators; seed should create 3+ users, 4+ teams, 3+ activities, 3+ workouts idempotently.
  - **If failing**: Check model schemas for missing validators or seed logic (upsert conditions).

- **API command**: `npm run test:api --prefix octofit-tracker/backend`
  - **Purpose**: Validate Express endpoints reject invalid input (4xx), create/list records, and compute derived fields.
  - **Expected on first run**: Failures in `createActivity`, `createTeam`, `getLeaderboard`, `suggestWorkouts` service stubs.
  - **Failing tests explain**:
    - Activities: `createActivity` throws 501; needs validation, ObjectId checks, points computation, occurredAt assignment.
    - Teams: `createTeam` throws 501; needs name validation.
    - Leaderboard: `getLeaderboard` returns []; needs aggregation pipeline.
    - Suggestions: `suggestWorkouts` returns any workout; needs email lookup, activity pattern matching, focus alignment.

- **Frontend command**: `npm test --prefix octofit-tracker/frontend`
  - **Purpose**: Validate React components fetch from `/api`, handle loading/error/empty states, render data with accessibility.
  - **Expected on first run**: Failures in components that fetch but lack state management or error handling.
  - **Failing tests explain**:
    - Each route must announce loading state: `<div role="status">` with "loading" text.
    - Each route must show empty state if API returns empty array.
    - Each route must announce error as an alert: `<div role="alert">` if API fails.

- **Backend build command**: `npm run build --prefix octofit-tracker/backend`
  - **Purpose**: TypeScript compilation and type checking.
  - **Expected**: No errors after services are implemented.

- **Frontend build command**: `npm run build --prefix octofit-tracker/frontend`
  - **Purpose**: Vite bundle and type checking.
  - **Expected**: No errors after fetch logic is wired.

## Failing tests summary

| Tier | Test file | Issue | Blocker |
| --- | --- | --- | --- |
| Data | `data.contract.test.ts` | Models may need validators; seed likely incomplete | Phase 1 |
| API | `api.contract.test.ts` | Service stubs throw 501; need validation, aggregation, matching logic | Phase 2 |
| Presentation | `app.contract.test.tsx` | Fetch logic or state management missing | Phase 3 |
