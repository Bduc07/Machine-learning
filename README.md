# Google Auth + Neon Postgres + React (Vite)

A minimal, production-shaped example of "Sign in with Google" for a React
(Vite) frontend, backed by an Express API that verifies the Google ID token
and stores/looks up the user in a [Neon](https://neon.tech) Postgres database,
issuing an httpOnly session cookie (JWT) for subsequent requests.

```
google-auth-neon-app/
├── client/          React + Vite frontend
└── server/          Express API + Neon Postgres
```

## How it works

1. The frontend renders Google's official `GoogleLogin` button
   (`@react-oauth/google`, which wraps Google Identity Services).
2. On success, Google hands the frontend a signed **ID token** (JWT). The
   frontend POSTs it to `POST /api/auth/google`.
3. The backend verifies that ID token against Google's public keys
   (`google-auth-library`), extracting the user's Google ID, email, name,
   and picture.
4. The backend **upserts** the user into a `users` table in Neon Postgres,
   then signs its own short-lived JWT and sets it as an `httpOnly` cookie.
   This is the app's own session — the browser never sees the Google token
   again.
5. `GET /api/auth/me` reads that cookie on page load to restore the session;
   `POST /api/auth/logout` clears it.

Because verification happens server-side, the database is only ever touched
with a token Google has already vouched for — the frontend can't forge a
user.

## 1. Create a Google OAuth Client ID

1. Go to the [Google Cloud Console](https://console.cloud.google.com/apis/credentials).
2. Create a project (or pick an existing one).
3. **OAuth consent screen**: set it up (External is fine for testing), add
   your email as a test user if it's still in "Testing" mode.
4. **Credentials → Create Credentials → OAuth client ID**:
   - Application type: **Web application**
   - Authorized JavaScript origins: `http://localhost:5173`
   - Authorized redirect URIs: not required for this flow (Google Identity
     Services uses a popup/One Tap, not a redirect)
5. Copy the generated **Client ID** — you'll need it in both `client/.env`
   and `server/.env`.

## 2. Create a Neon Postgres database

1. Sign up / log in at [neon.tech](https://neon.tech) and create a project.
2. On the project dashboard, copy the **connection string** (it looks like
   `postgresql://<user>:<password>@<endpoint>.neon.tech/<db>?sslmode=require`).
3. Run the schema in `server/db/schema.sql` against that database. Easiest
   way is Neon's SQL editor in the web console — paste the file contents and
   run it. (Or `psql "<connection string>" -f server/db/schema.sql` if you
   have `psql` installed locally.)

## 3. Configure environment variables

```bash
cp server/.env.example server/.env
cp client/.env.example client/.env
```

`server/.env`:

```
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
DATABASE_URL=postgresql://user:password@ep-xxxx.neon.tech/neondb?sslmode=require
JWT_SECRET=replace-with-a-long-random-string
CLIENT_URL=http://localhost:5173
PORT=5000
```

`client/.env`:

```
VITE_GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
```

(Same Google Client ID in both — the frontend uses it to render the button,
the backend uses it to verify the token's audience.)

## 4. Install and run

```bash
# backend
cd server
npm install
npm run dev      # http://localhost:5000

# frontend, in a second terminal
cd client
npm install
npm run dev       # http://localhost:5173
```

Open `http://localhost:5173`, click "Sign in with Google", and you should
see your profile (name, email, avatar) pulled back from `/api/auth/me`.

The Vite dev server proxies `/api/*` to `http://localhost:5000` (see
`client/vite.config.js`), so the frontend and backend behave as same-origin
in development and cookies work without extra CORS configuration. In
production, either serve both from the same domain behind a reverse proxy,
or set `VITE_API_URL` on the client and configure `CLIENT_URL` /
`cookie.sameSite`/`secure` on the server for your real domains (notes are
in `server/index.js`).

## Notes on going to production

- Set `NODE_ENV=production`, serve the app over HTTPS, and change the auth
  cookie to `secure: true, sameSite: 'none'` if the frontend and API are on
  different domains (see `server/routes/auth.js`).
- Rotate `JWT_SECRET` to a long random value and keep it out of source
  control.
- Neon connection pooling: for serverless/edge deployments, consider Neon's
  pooled connection string (`-pooler` in the hostname) or the
  `@neondatabase/serverless` driver instead of plain `pg`.
- Add rate limiting / brute-force protection on `/api/auth/google` if you
  expose this publicly.
