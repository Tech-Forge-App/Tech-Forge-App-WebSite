---
title: "Chapter 4 — Protecting an endpoint"
description: Lock down a route so only requests carrying a valid Keycloak token get through, and read the caller's identity straight from the token.
sidebar:
  order: 4
  badge:
    text: New
    variant: note
---

In [Chapter 3](/tutorial/03-authenticating-a-user/) you logged a user in and got
back an **access token**. So far that token has been a key without a lock. In
this chapter you'll add the lock: a **protected endpoint**, `GET /auth/me`, that
**rejects** any request without a valid token — and, for valid ones, tells you
exactly who's calling. 🛡️

## What you'll build

By the end of this chapter you'll have:

- A clear picture of how a NestJS **guard** protects a route.
- An understanding of how the API **verifies** a token (signature, issuer,
  expiry) — without ever calling back to Keycloak on each request.
- A working `GET /auth/me` endpoint that returns the caller's identity, and a
  feel for how it rejects bad or missing tokens.

## What is the API doing?

When a request hits a protected route, a **guard** runs *before* your handler:

```
Request ──▶ JwtAuthGuard ──▶  handler (only if the token checks out)
               │
               └─ no/invalid token ──▶ 401 Unauthorized
```

The guard:

1. Reads the token from the `Authorization: Bearer <token>` header.
2. **Verifies** it: checks the cryptographic **signature**, the **issuer**, and
   the **expiry**.
3. Derives the caller's **identity** from the token's claims and attaches it to
   the request.
4. Lets the request through — or throws `401 Unauthorized`.

:::tip[Verify, don't ask]
The API doesn't phone Keycloak on every request. A Keycloak token is a **signed
JWT**: anyone holding Keycloak's *public* key can confirm it's genuine and
untampered, entirely offline. The API fetches those public keys once from
Keycloak's **JWKS** endpoint (`.../protocol/openid-connect/certs`), caches them,
and verifies signatures locally. That's what makes token auth fast and scalable.
:::

:::note[What's a "claim"?]
A JWT's payload is a set of **claims** — facts Keycloak asserts about the token:
`sub` (the user id), `preferred_username`, `email`, `realm_access.roles`, `exp`
(expiry), `iss` (issuer), and more. Once the signature is verified, the API
*trusts* these claims and builds the caller's identity from them. No database
lookup needed.
:::

## Prerequisites

Before you start, make sure:

- You completed [Chapter 3](/tutorial/03-authenticating-a-user/) and can log in
  with `POST /auth/login`.
- **Keycloak is running** and the **NestJS API is running**
  (`pnpm start:dev` in `Tech-Forge-App/api/`).

After restarting the API, the new protected route shows up in the logs next to
the others:

```
[RouterExplorer] Mapped {/auth/login, POST} route
[RouterExplorer] Mapped {/auth/me, GET} route
[RouterExplorer] Mapped {/users, POST} route
```

## Step 1 — Try the protected route *without* a token

Let's confirm the lock works before we hold the key. Call `GET /auth/me` with no
`Authorization` header:

```bash
curl -i http://localhost:3000/auth/me
```

You get **HTTP 401 Unauthorized**:

```json
{
  "message": "Token d'accès manquant",
  "error": "Unauthorized",
  "statusCode": 401
}
```

🔒 The guard stopped the request before it ever reached the handler.

## Step 2 — Get a fresh token

Log in again (tokens last only 5 minutes) and capture the access token:

```bash
ACCESS_TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdupont","password":"S3cret-pa55phrase"}' \
  | jq -r .accessToken)
```

## Step 3 — Call the protected route *with* the token

Now send the same request, but attach the token:

```bash
curl http://localhost:3000/auth/me \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

This time you get **HTTP 200 OK** with your identity, read straight from the
verified token:

```json
{
  "id": "6f3b1c2a-2d4e-4f5a-8b6c-7d8e9f0a1b2c",
  "username": "jdupont",
  "email": "jean.dupont@example.com",
  "roles": ["user"]
}
```

🎉 The guard verified the signature, confirmed the issuer and expiry, pulled the
claims, and handed your handler a ready-to-use identity.

### The response fields

| Field      | Source claim              | Meaning                              |
| ---------- | ------------------------- | ------------------------------------ |
| `id`       | `sub`                     | The user's Keycloak id               |
| `username` | `preferred_username`      | The user's username                  |
| `email`    | `email`                   | The user's email (if present)        |
| `roles`    | `realm_access.roles`      | The realm roles the token grants     |

:::note[Where do `roles` come from?]
The freshly imported realm gives application users no roles by default, so you
may see `"roles": []`. To see a role appear, open the admin console →
**Users** → **jdupont** → **Role mapping** → assign the `user` role, then log in
again to get a token that carries it.
:::

## Step 4 — See the guard reject a *bad* token

Verification is real — let's prove it. Tamper with the token by flipping its last
character and try again:

```bash
curl -i http://localhost:3000/auth/me \
  -H "Authorization: Bearer ${ACCESS_TOKEN}x"
```

→ **401 Unauthorized**. The signature no longer matches, so the guard refuses it:

```json
{
  "message": "Token d'accès invalide ou expiré",
  "error": "Unauthorized",
  "statusCode": 401
}
```

The same `401` is what you'd get from an **expired** token (wait 5 minutes and
retry the Step 3 call) or a token issued by a **different realm**.

| Status | Meaning                                                       |
| ------ | ------------------------------------------------------------- |
| `200`  | Token valid; identity returned                                |
| `401`  | Token missing, malformed, expired, wrong issuer, or tampered  |

## How it works in the code

A few small pieces fit together. You don't have to write them — they ship with
the API — but it's worth seeing how thin each one is.

**The guard** reads the header, verifies the token, and attaches the identity:

```ts
// src/auth/jwt-auth.guard.ts (essence)
const token = this.extractBearerToken(request.headers.authorization);
if (!token) throw new UnauthorizedException("Token d'accès manquant");
try {
  request.user = await this.verifier.verify(token);
} catch {
  throw new UnauthorizedException("Token d'accès invalide ou expiré");
}
return true;
```

**The verifier** checks the signature against the realm's public keys plus the
issuer and expiry, then maps the claims to a tidy identity:

```ts
// src/auth/keycloak-token-verifier.ts (essence)
const { payload } = await jwtVerify(token, this.keys, { issuer: this.issuer });
return {
  id: String(payload.sub),
  username: payload.preferred_username ?? '',
  email: payload.email,
  roles: payload.realm_access?.roles ?? [],
};
```

**The route** opts in to the guard and pulls the identity from the request with a
small `@CurrentUser()` decorator:

```ts
// src/auth/auth.controller.ts (essence)
@Get('me')
@UseGuards(JwtAuthGuard)
me(@CurrentUser() user: AuthenticatedUser): MeResponse {
  return user;
}
```

To protect **any** future route, you add the same `@UseGuards(JwtAuthGuard)` and
ask for `@CurrentUser()`. That's the whole pattern.

:::tip[Contract-first, still]
`GET /auth/me` and its `MeResponse` shape live in `api/openapi/openapi.yaml`
first — including a `bearerAuth` **security scheme** that documents the
`Authorization: Bearer` requirement. The TypeScript types come from there via
`pnpm gen:api`, exactly like every other endpoint in this tutorial.
:::

## Troubleshooting

- **Always `401`, even right after logging in** — your token may already be
  expired (they last 5 minutes); grab a fresh one (Step 2). Also check you wrote
  `Authorization: Bearer <token>` with a single space.
- **`502`-style errors / the API hangs on first protected call** — the API
  fetches Keycloak's public keys on demand; if Keycloak is down or
  `KEYCLOAK_BASE_URL` is wrong, verification can't complete. Make sure the stack
  is up.
- **`roles` is always empty** — that's expected until you map a role to the user
  (see the note in Step 3).
- **`jq: command not found`** — install `jq`, or copy the `accessToken` value out
  of the login response by hand into the `Authorization` header.

## What you've learned

In this chapter you:

- ✅ Saw how a NestJS **guard** protects a route before the handler runs.
- ✅ Learned that the API **verifies** JWTs offline using Keycloak's public keys.
- ✅ Called `GET /auth/me` with a token and read your identity from its claims.
- ✅ Watched the guard reject missing, tampered, and expired tokens with `401`.
- ✅ Met the reusable `@UseGuards(JwtAuthGuard)` + `@CurrentUser()` pattern.

## Next steps

You've gone the full loop: **create** a user, **authenticate** them, and
**protect** a route with the token they receive. From here, natural next steps
are **role-based authorization** (only `admin`s may call certain routes),
**refresh tokens** (staying logged in without re-entering a password), and
swapping the password grant for the **Authorization Code flow** in a real
frontend.

---

*This is Chapter 4 of the TechForge newcomer tutorial. More chapters may follow!*
