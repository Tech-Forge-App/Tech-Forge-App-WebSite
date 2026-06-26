---
title: "Chapter 2 — Running the NestJS API"
description: Connect the contract-first NestJS API to Keycloak and create your first user through a real HTTP endpoint.
sidebar:
  order: 2
  badge:
    text: New
    variant: note
---

In [Chapter 1](/tutorial/01-keycloak-setup/) you started **Keycloak** and grabbed
your first access token by hand. In this chapter you'll run the **TechForge
NestJS API** — the application that talks to Keycloak for you — and create a real
user account with a single HTTP request. 🚀

No NestJS experience is required. We'll explain the moving parts as we meet them.

## What you'll build

By the end of this chapter you'll have:

- The NestJS API running locally on port `3000`.
- A clear picture of how the API authenticates to Keycloak on your behalf.
- A brand-new user created in the `techforge` realm via `POST /users`.

## What is the API doing?

The API sits between you (or a frontend) and Keycloak:

```
You / Frontend  ──HTTP──▶  NestJS API  ──Admin REST API──▶  Keycloak
```

Instead of crafting Keycloak admin calls yourself, you send a simple request to
the API, and it:

1. Authenticates to Keycloak using the **client credentials** grant — the exact
   same call you made by hand in Chapter 1, but now automated.
2. Uses the resulting token to call Keycloak's **Admin REST API** and create the
   user.
3. Returns a clean, predictable response to you.

:::tip[Contract-first]
The API is built **contract-first**: the file `api/openapi/openapi.yaml` is the
single source of truth. The TypeScript types the server uses are *generated* from
it (`pnpm gen:api`). When you change the contract, you regenerate the types and
the compiler tells you what to update. The shape of every request and response in
this chapter comes straight from that contract.
:::

## Prerequisites

Before you start, make sure:

- You completed [Chapter 1](/tutorial/01-keycloak-setup/) and **Keycloak is
  running** (`docker compose up -d` in `Tech-Forge-App/`). Quick check:

  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" \
    http://localhost:8080/realms/techforge/.well-known/openid-configuration
  ```

  This should print `200`.
- **Node.js** (version 20 or newer) and **pnpm** are installed
  (`node --version` and `pnpm --version` should both print a version).
- A terminal open in the `Tech-Forge-App/api/` directory.

:::note[Why pnpm?]
TechForge uses **pnpm** as its package manager — it's fast and disk-efficient. If
you don't have it yet, the simplest way is `corepack enable` (bundled with
Node.js), or follow the [pnpm install guide](https://pnpm.io/installation).
:::

## Step 1 — Install dependencies

From the `Tech-Forge-App/api/` directory:

```bash
pnpm install
```

This downloads NestJS and the other libraries the API needs.

## Step 2 — Create your environment file

Like the Keycloak stack, the API reads its settings from a `.env` file. Copy the
template:

```bash
cp .env.example .env
```

The defaults already match the Keycloak setup from Chapter 1, so you don't need
to change anything for a first local run:

```bash
# HTTP port of the NestJS API
PORT=3000

# Keycloak — Admin REST API
KEYCLOAK_BASE_URL=http://localhost:8080
KEYCLOAK_REALM=techforge
KEYCLOAK_CLIENT_ID=techforge-api
KEYCLOAK_CLIENT_SECRET=techforge-api-secret
```

:::caution[Same secret, same warning]
`techforge-api-secret` is a development-only secret baked into the imported
realm. In a real deployment you'd generate a fresh client secret in Keycloak and
keep it out of source control.
:::

## Step 3 — Generate the API types from the contract

Because the project is contract-first, regenerate the TypeScript types from
`openapi/openapi.yaml` before you build or run:

```bash
pnpm gen:api
```

This writes `src/generated/api.d.ts`. You never edit that file by hand — it's
produced from the contract.

:::note
You only need to re-run this when the contract changes. We run it now so your
checkout is in a known-good state.
:::

## Step 4 — Start the API

Launch the API in watch mode (it restarts automatically when you edit a file):

```bash
pnpm start:dev
```

Watch the logs until you see a line confirming Nest mapped the route:

```
[RouterExplorer] Mapped {/users, POST} route
[NestApplication] Nest application successfully started
```

The API is now listening on <http://localhost:3000>.

:::caution[Port 3000 already in use?]
Another app (a frontend dev server, for instance) might already hold port 3000.
If startup fails with `EADDRINUSE`, set a different port in `.env`, e.g.
`PORT=3100`, and restart. Adjust the URLs in the rest of this chapter
accordingly.
:::

## Step 5 — Create your first user

This is the moment everything has been leading to. Send a `POST` request to the
`/users` endpoint:

```bash
curl -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "jdupont",
    "email": "jean.dupont@example.com",
    "firstName": "Jean",
    "lastName": "Dupont",
    "password": "S3cret-pa55phrase"
  }'
```

If all is well, you get back **HTTP 201 Created** with the new user's Keycloak id:

```json
{
  "id": "6f3b1c2a-2d4e-4f5a-8b6c-7d8e9f0a1b2c",
  "username": "jdupont",
  "email": "jean.dupont@example.com"
}
```

🎉 Behind the scenes the API just obtained a token, called Keycloak's Admin REST
API, and read the new user's id from the response.

### The request fields

| Field       | Required | Rules                                  |
| ----------- | -------- | -------------------------------------- |
| `username`  | ✅       | 3–60 characters                        |
| `email`     | ✅       | must be a valid email address          |
| `password`  | ✅       | 8–128 characters                       |
| `firstName` | ➖       | optional, up to 100 characters         |
| `lastName`  | ➖       | optional, up to 100 characters         |

## Step 6 — See it in Keycloak

Confirm the user really exists. Open the admin console (from Chapter 1):

👉 <http://localhost:8080/admin> → realm **techforge** → **Users**

You should see `jdupont` in the list. You can also query it from the command
line, reusing the token trick from Chapter 1:

```bash
TOKEN=$(curl -s -X POST \
  http://localhost:8080/realms/techforge/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=techforge-api \
  -d client_secret=techforge-api-secret | jq -r .access_token)

curl -s "http://localhost:8080/admin/realms/techforge/users?username=jdupont&exact=true" \
  -H "Authorization: Bearer $TOKEN" | jq
```

## Step 7 — What happens when things go wrong?

The API validates your request against the contract and reports errors clearly.
Try these on purpose to see how it behaves:

**Invalid body** (bad email, password too short) → **400 Bad Request**:

```bash
curl -i -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"jd","email":"not-an-email","password":"short"}'
```

```json
{
  "message": [
    "username must be longer than or equal to 3 characters",
    "email must be an email",
    "password must be longer than or equal to 8 characters"
  ],
  "error": "Bad Request",
  "statusCode": 400
}
```

**Unknown field** (the contract is strict) → **400 Bad Request**:

```bash
curl -i -X POST http://localhost:3000/users \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont","email":"jean.dupont@example.com","password":"S3cret-pa55phrase","isAdmin":true}'
```

→ `property isAdmin should not exist`

**Duplicate user** (run the Step 5 request twice) → **409 Conflict**:

```json
{
  "message": "Nom d'utilisateur ou e-mail déjà utilisé",
  "error": "Conflict",
  "statusCode": 409
}
```

| Status | Meaning                                             |
| ------ | --------------------------------------------------- |
| `201`  | User created                                        |
| `400`  | Invalid body or unknown field                       |
| `409`  | Username or email already in use                    |
| `502`  | The API could not talk to Keycloak                  |

## Troubleshooting

- **`502 Bad Gateway`** — the API can't reach Keycloak. Make sure the Keycloak
  stack is running (`docker compose ps` in `Tech-Forge-App/`) and that
  `KEYCLOAK_BASE_URL` in `.env` is correct.
- **`EADDRINUSE` on startup** — port 3000 is taken; set `PORT=3100` in `.env`
  (see Step 4).
- **Every login returns `invalid_client`** — check `KEYCLOAK_CLIENT_ID` /
  `KEYCLOAK_CLIENT_SECRET` and that you're pointing at the `techforge` realm.
- **Types out of date after editing the contract** — re-run `pnpm gen:api`.

## What you've learned

In this chapter you:

- ✅ Understood how the API brokers requests between you and Keycloak.
- ✅ Installed, configured, and started the NestJS API with pnpm.
- ✅ Regenerated the server types from the OpenAPI contract.
- ✅ Created a real user with `POST /users` and verified it in Keycloak.
- ✅ Saw how validation and conflict errors are reported.

## Next steps

You now have a working API that can create accounts. A future chapter will build
on this to **read, update, and authenticate** users — and to protect endpoints so
only valid tokens get through.

---

*This is Chapter 2 of the TechForge newcomer tutorial. More chapters are on the
way!*
