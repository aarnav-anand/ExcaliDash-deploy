---
title: Deploy ExcaliDash on Render and Neon
description: Run a personal ExcaliDash workspace on Render's free tier with durable Neon PostgreSQL storage.
---

# Deploy ExcaliDash on Render and Neon

This deployment is for a small personal workspace. It uses a free Render static
site for the frontend, a free Render web service for the API and real-time
collaboration server, and Neon PostgreSQL for persistent data.

The backend may sleep after inactivity. The first request after it sleeps can
take about a minute. Drawings, accounts, sharing settings, version history, and
image records survive backend restarts because they live in Neon, not on the
Render filesystem.

::: warning Free-tier limits
Render's free web service has limited memory and can restart or sleep. Neon
Free has a storage limit. This is suitable for a personal or small hobby
workspace, not for a service requiring an uptime guarantee. Keep exports and
database backups outside these providers.
:::

## 1. Create the Neon database

1. Create a Neon project in the region nearest your users.
2. Open **Connect**, turn connection pooling off, and copy the **direct**
   PostgreSQL connection string. Keep `sslmode=require` if Neon includes it.
3. Do not expose that value in Git, the frontend, or a client-side environment
   variable. It belongs only in the Render backend's `DATABASE_URL`.

ExcaliDash stores drawing data, accounts, access controls, history, and image
blobs in this database when S3 is not configured. The free database limit means
this deployment is best for drawings and modest image uploads, not a large
asset archive.

## 2. Create the Render services

Push this repository, including `render.yaml`, to a GitHub repository you own.
In Render, select **New > Blueprint**, connect the repository, and create the
Blueprint. Render creates two public services: `excalidash-backend` for Express
and Socket.IO, and `excalidash-frontend` for the Vite static site.

When Render asks for `DATABASE_URL`, paste the Neon direct connection string.
For the two URL variables, use the values in the next step. If Render requires
values before it displays URLs, enter temporary HTTPS placeholders, create the
services, then update the variables immediately and redeploy both services.

Do not create a Render PostgreSQL service. Its free database is temporary; the
Blueprint is intentionally configured for Neon.

## 3. Connect the frontend and backend

Copy both Render service URLs from their dashboard pages. They resemble:

```text
https://excalidash-frontend.onrender.com
https://excalidash-backend.onrender.com
```

Set the frontend service's `VITE_API_URL` to the backend URL. Set the backend
service's `FRONTEND_URL` to the frontend URL. Then manually deploy the backend,
wait for `/health` to return `200`, and manually deploy the frontend.

`VITE_API_URL` is compiled into the static site, so changing it has no effect
until the frontend is rebuilt. Use the full backend URL, with neither a trailing
slash nor `/api`. Docker Compose uses nginx to map `/api` to the backend; this
split hosting deployment connects the browser directly to the backend. The
application also uses that same URL for Socket.IO collaboration.

The two default `onrender.com` hosts are same-site, so local-auth cookies work
with the project's secure `SameSite=Lax` cookie settings. If you add custom
domains, use subdomains of the same registered domain, such as
`draw.example.com` and `api.draw.example.com`; then update both variables and
redeploy.

## 4. Confirm the database migration and health check

Open the backend's Render logs. On the first successful deployment, confirm:

1. `Copying postgresql migrations...`
2. `Running database migrations...`
3. A healthy `/health` check.
4. A line beginning `[BOOTSTRAP SETUP] One-time admin setup code:`.

The setup code is deliberately printed only in backend logs. It expires after
15 minutes and is consumed once. If it expires, redeploy the backend to issue a
fresh code. Do not share the code or paste it into a public issue.

## 5. Create the first administrator safely

Open the frontend URL. Choose the local-account setup flow and create the first
administrator using the one-time code from the backend log.

Use a unique email address and a long password that satisfies the default
policy: at least 12 characters with uppercase, lowercase, number, and symbol.
Save the password in a password manager. The successful setup consumes the
bootstrap code and disables the bootstrap-only account.

After the first administrator exists:

1. Open **Admin** and leave public registration disabled unless you explicitly
   want anyone with the URL to create an account.
2. Use **New user** to create named accounts for trusted collaborators.
3. Give collaborators access only through drawing or collection sharing; do
   not share the administrator account.
4. Create a non-critical test drawing, reload it, and open it in a second
   signed-in browser session to confirm persistence and live collaboration.

Never set `AUTH_MODE=disabled` on this public deployment. That mode gives every
visitor the same identity and access.

## 6. Create and organize the workspace

The first administrator can create drawings immediately from the dashboard.
Use collections to organize drawings and the share controls to grant either
view or edit access. For a personal workspace, keep every drawing private by
default and share individual drawings only when needed.

For long-term safety:

- Export important drawings and your library periodically.
- Keep a copy of the generated Render secrets only through Render's protected
  environment-variable storage; do not rotate `JWT_SECRET`, `CSRF_SECRET`, or
  `API_KEY_HASH_PEPPER` casually.
- Monitor Neon storage. Image uploads and historical snapshots count toward
  database usage.
- Before upgrades, export important work and confirm the backend health check
  after the automatic redeploy.

## Troubleshooting

### Login or collaboration fails after deployment

Check the exact URLs. `FRONTEND_URL` must equal the browser-visible frontend
origin, while `VITE_API_URL` must equal the backend origin. Redeploy both after
changing either value. Also verify `TRUST_PROXY=1` remains set on the backend.

### First page load is slow

The Render backend likely slept. Wait for the service to wake, reload the page,
and sign in. This does not lose Neon-backed data.

### No administrator setup code appears

Confirm `AUTH_MODE=local`, then inspect the first backend start after the
database was initialized. The code appears only while there are no active users.
If it expired before use, redeploy the backend to create a new code.

### Backend cannot connect to Neon

Verify that `DATABASE_URL` is the Neon direct PostgreSQL connection string and
includes its SSL setting. Do not use a SQLite `file:` URL or expose the URL to
the frontend.
