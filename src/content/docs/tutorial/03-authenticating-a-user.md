---
title: "Chapter 3 — Authenticating a user"
description: Log a user in through the NestJS API and get back an access token you can use to call the rest of the API.
sidebar:
  order: 3
---

In [Chapter 2](/tutorial/02-nestjs-api/) you created a real user with
`POST /users`. But creating an account is only half the story — that user now
needs to **prove who they are** and obtain a token. In this chapter you'll log in
through the API's new `POST /auth/login` endpoint and get back an **access token**
to authorize the rest of your API calls. 🔑

## What you'll build

By the end of this chapter you'll have:

- A clear picture of how the API turns a username + password into a token.
- A real access token issued by Keycloak for the user you created in Chapter 2.
- An understanding of what's inside that token and how you'll use it.

## What is the API doing?

Just like user creation, the API brokers the login for you:

```
You / Frontend  ──HTTP──▶  NestJS API  ──Token endpoint──▶  Keycloak
```

When you send credentials to `POST /auth/login`, the API:

1. Calls Keycloak's **token endpoint** using the *password* grant (the
   "Resource Owner Password Credentials" flow) of the `techforge-api` client.
2. Lets Keycloak verify the username and password against the `techforge` realm.
3. Returns a clean token payload to you — the `accessToken` is the one that
   matters.

:::tip[Two different grants]
In Chapter 2 the API used the **client credentials** grant to authenticate
*itself* (the service account) so it could manage users. Here it uses the
**password** grant to authenticate *a user*. Same token endpoint, different
purpose: one is "the API proving it's the API", the other is "a person proving
they're that person".
:::

:::caution[About the password grant]
The password grant is convenient for a first-party app and a tutorial, but the
OAuth 2.0 working group discourages it for production: the user's password flows
through your backend. A real frontend would normally use the **Authorization
Code flow with PKCE** and never let the password touch the API. We use the
password grant here to keep the focus on the API. A later chapter will revisit
this.
:::

## Prerequisites

Before you start, make sure:

- You completed [Chapter 2](/tutorial/02-nestjs-api/), **Keycloak is running**,
  and the **NestJS API is running** (`pnpm start:dev` in `Tech-Forge-App/api/`).
- You **created the `jdupont` user** in Chapter 2 (Step 5). If you didn't, go
  back and run that request now — you need a real user to log in with.
- `jq` is handy for the token-inspection steps but not required.

After restarting the API, watch the logs for the new route alongside the old one:

```
[RouterExplorer] Mapped {/auth/login, POST} route
[RouterExplorer] Mapped {/users, POST} route
```

## Step 1 — Log in and get a token

Send the user's credentials to the login endpoint:

```bash
curl -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "jdupont",
    "password": "S3cret-pa55phrase"
  }'
```

On success you get **HTTP 200 OK** with the token payload:

```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUI...",
  "expiresIn": 300,
  "refreshToken": "eyJhbGciOiJIUzUxMiIsInR5cCIgOiAiSldUI...",
  "refreshExpiresIn": 1800,
  "tokenType": "Bearer"
}
```

🎉 The user is authenticated. The `accessToken` is a **JWT** valid for
`expiresIn` seconds (300 = 5 minutes, matching the realm config).

### The response fields

| Field              | Meaning                                                        |
| ------------------ | -------------------------------------------------------------- |
| `accessToken`      | The JWT you send on every authorized request                   |
| `expiresIn`        | Access-token lifetime, in seconds                              |
| `refreshToken`     | Used to get a fresh access token without logging in again      |
| `refreshExpiresIn` | Refresh-token lifetime, in seconds                             |
| `tokenType`        | Always `Bearer`                                                |

### The request fields

| Field      | Required | Rules                          |
| ---------- | -------- | ------------------------------ |
| `username` | ✅       | the user's username or email   |
| `password` | ✅       | the user's password            |

## Step 2 — Capture the token in a variable

To reuse it in later requests, store the `accessToken` in a shell variable:

```bash
ACCESS_TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont","password":"S3cret-pa55phrase"}' \
  | jq -r .accessToken)

echo "$ACCESS_TOKEN"
```

You should see a long, three-part string separated by dots — that's the JWT.

## Step 3 — Look inside the token

A JWT is not encrypted, just **signed** — anyone can read its contents (but only
Keycloak can produce a valid signature). Decode the middle part (the payload) to
see who it's for:

```bash
echo "$ACCESS_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq
```

You'll see claims like:

```json
{
  "exp": 1751280000,
  "iss": "http://localhost:8080/realms/techforge",
  "preferred_username": "jdupont",
  "email": "jean.dupont@example.com",
  "realm_access": { "roles": ["user"] }
}
```

- `iss` — who issued the token (your Keycloak realm).
- `preferred_username` / `email` — *who* the token represents.
- `realm_access.roles` — what the user is allowed to do.
- `exp` — when the token expires (Unix timestamp).

This is exactly what a protected endpoint will inspect to decide whether to let a
request through.

## Step 4 — Use the token to call the API

This is the whole point of authenticating: the `accessToken` is what you attach
to any request that needs to know *who you are*. You send it in the standard
`Authorization` header:

```bash
curl http://localhost:3000/some-protected-route \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

:::note[A protected route is waiting]
In [Chapter 4](/tutorial/04-protecting-an-endpoint/) you'll meet
`GET /auth/me` — the first endpoint that **requires** a valid token and reads the
caller's identity straight from it. Keep this token (or the login command) handy:
you'll point that `Authorization` header at it and watch the API accept your
request — and reject one without a token.
:::

## Step 5 — What happens when things go wrong?

The API validates your request against the contract and reports failures clearly.

**Wrong password** (or unknown user) → **401 Unauthorized**:

```bash
curl -i -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont","password":"not-my-password"}'
```

```json
{
  "message": "Identifiants invalides",
  "error": "Unauthorized",
  "statusCode": 401
}
```

**Missing field** (no password) → **400 Bad Request**:

```bash
curl -i -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont"}'
```

**Unknown field** (the contract is strict) → **400 Bad Request**:

```bash
curl -i -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont","password":"S3cret-pa55phrase","realm":"evil"}'
```

| Status | Meaning                                            |
| ------ | -------------------------------------------------- |
| `200`  | Authenticated; token returned                      |
| `400`  | Invalid body or unknown field                      |
| `401`  | Wrong username or password                         |
| `502`  | The API could not talk to Keycloak                 |

## Troubleshooting

- **`401 Unauthorized` even with the right password** — confirm you created the
  `jdupont` user in Chapter 2 and that you're hitting the `techforge` realm. A
  freshly imported realm has no application users until you add them.
- **`502 Bad Gateway`** — the API can't reach Keycloak. Check the stack is up
  (`docker compose ps` in `Tech-Forge-App/`) and `KEYCLOAK_BASE_URL` in `.env`.
- **`invalid_client` in the API logs** — the `techforge-api` client secret or id
  is wrong; check `KEYCLOAK_CLIENT_ID` / `KEYCLOAK_CLIENT_SECRET`.
- **`Account is not fully set up` / `Account disabled`** — the user exists but
  isn't enabled, or has a required action pending in Keycloak. Open the admin
  console → **Users** → **jdupont** and clear any required actions.
- **`base64: invalid input` when decoding the token** — some `base64` builds need
  `base64 -D` (macOS) or padding; the decode step is optional, the token still
  works.

## What you've learned

In this chapter you:

- ✅ Logged a user in through `POST /auth/login` and received an access token.
- ✅ Saw how the API uses Keycloak's password grant on the user's behalf.
- ✅ Inspected the JWT and read the identity and roles inside it.
- ✅ Learned how to attach the token via the `Authorization: Bearer` header.
- ✅ Saw how invalid credentials and bad requests are reported.

## Next steps

You can now **create** users and **authenticate** them. The missing piece is
**protecting** endpoints — making the API reject requests that don't carry a
valid token, and reading the caller's identity from the ones that do. That's
[Chapter 4](/tutorial/04-protecting-an-endpoint/).

---

*This is Chapter 3 of the TechForge newcomer tutorial. More chapters are on the
way!*
