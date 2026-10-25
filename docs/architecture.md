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

- Presentation tier decision: Keep React components focused on route rendering and visible states, with `src/api/client.ts` responsible for API URL construction and fetch behavior. Use React Router for the four capability views and accessible loading, empty, success, and error text.
- Logic tier decision: Keep `/api/*` route wiring in the Express router and validation, aggregation, and recommendation behavior in explicit service functions. Return deliberate `4xx` responses for client or reference errors, rather than allowing Mongoose cast or missing-data failures to become `500` responses.
- Data tier decision: Define required Mongoose schema fields for users, teams, activities, and workouts. Store stable demo records with upserts, preserving unrelated user data and updating an existing demo user's previous team membership when its team changes.
- Activity tracking approach: Validate activity input and object IDs before querying, verify that the referenced user and team exist, then create an activity with computed points and an `occurredAt` timestamp. `GET` lists persisted activities through the service layer.
- Team management approach: Reject teams without names in service validation and rely on Mongoose validation for required mascot and member data. Create teams with submitted names, mascots, and member IDs, and expose the resulting records through list and create routes.
- Leaderboard approach: Aggregate activities by team and sum each team's points. Sort descending, assign a 1-based rank, and return only `teamName`, `totalPoints`, and `rank` through the leaderboard service.
- Workout suggestions approach: Resolve the URL-decoded user email to a seeded user and inspect the user's recent activity and fitness goal. Return matching workouts first with a reason tied to activity or goal, and return a deliberate `404` for an unknown user.

## Test plan

Run the narrow contracts in dependency order: data first, API second, then presentation. A data failure means the schemas or seed are incomplete; an API failure means route/service behavior is not satisfying the contracts; a frontend failure means rendering, shared API usage, or accessible states need correction. Build commands verify TypeScript and Vite integration after contract behavior passes.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` - validates Mongoose fields and idempotent demo seeding. Failure indicates missing required schema validation or seed relationships.
- API command: `npm run test:api --prefix octofit-tracker/backend` - validates activity, team, leaderboard, and workout API contracts. Failure indicates missing service behavior, invalid input handling, aggregation, or recommendations.
- Frontend command: `npm test --prefix octofit-tracker/frontend` - validates routed presentation, shared API usage, data rendering, and loading/empty/error states. Failure indicates incomplete components or incorrect user-visible behavior.
- Backend build command: `npm run build --prefix octofit-tracker/backend` - compiles the Express and Mongoose TypeScript implementation. Failure indicates a type or build integration problem.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` - compiles the React/Vite application. Failure indicates a component, route, import, or Vite configuration problem.
