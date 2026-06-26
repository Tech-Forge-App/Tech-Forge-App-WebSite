---
title: "Chapter 1 — Setting up Keycloak"
description: A beginner-friendly walkthrough to run the TechForge Keycloak authentication stack locally with Docker.
sidebar:
  order: 1
  badge:
    text: Start here
    variant: tip
---

Welcome to the **TechForge tutorial for newcomers**! 👋

This first chapter walks you through standing up **Keycloak**, the identity and
access-management server that powers authentication for the whole TechForge
platform. By the end you'll have a running login server, a pre-configured realm,
and a token you obtained yourself.

No prior Keycloak experience is required — we'll explain each concept as we go.

## What is Keycloak (and why do we use it)?

Keycloak is an open-source **Identity and Access Management** server. Instead of
each application building its own login screens, password storage, and token
logic, they delegate all of that to Keycloak.

A few terms you'll meet in this chapter:

- **Realm** — an isolated space that holds its own users, roles, and clients.
  TechForge uses a realm named `techforge`.
- **Client** — an application that talks to Keycloak. Our NestJS API is the
  client `techforge-api`.
- **Service account** — a non-human "robot" login attached to a client, used for
  server-to-server calls (here, so the API can create users on your behalf).
- **Token** — a short-lived signed string (a JWT) that proves who you are when
  calling a protected endpoint.

You don't need to memorise these — they'll click into place as you use them.

## Prerequisites

Before you start, make sure you have:

- **Docker** and **Docker Compose** installed and running
  (`docker --version` should print a version).
- A terminal open in the `Tech-Forge-App/` directory of the project.
- Port **8080** free on your machine (Keycloak's default web port).
- Optionally `curl` and `jq` to test tokens from the command line.

:::tip[New to Docker?]
Docker lets us run Keycloak and its database as ready-made containers, so you
don't have to install Java or PostgreSQL by hand. The provided
`docker-compose.yml` describes everything for you.
:::

## Step 1 — Create your environment file

The stack reads its secrets and ports from a `.env` file. A template is already
provided, so copy it:

```bash
cp .env.example .env
```

Open the new `.env` — for a first local run the defaults are fine:

```bash
# Keycloak admin (console + Admin REST API)
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=admin
KEYCLOAK_PORT=8080

# Keycloak database (Postgres)
KC_DB_NAME=keycloak
KC_DB_USER=keycloak
KC_DB_PASSWORD=keycloak
```

:::caution[These defaults are for local dev only]
`admin` / `admin` is fine on your laptop, but **never** use these credentials in
a shared or production environment. Change every password before deploying.
:::

## Step 2 — Start the stack

Launch Keycloak and its PostgreSQL database with a single command:

```bash
docker compose up -d
```

This starts two containers:

| Container                  | Role                                              |
| -------------------------- | ------------------------------------------------- |
| `techforge-keycloak-db`    | PostgreSQL 16 — stores Keycloak's data            |
| `techforge-keycloak`       | Keycloak 26.2 — the authentication server itself  |

The first boot takes about **30 seconds** while Keycloak runs database
migrations and imports the `techforge` realm. Follow the logs until it's ready:

```bash
docker compose logs -f keycloak
```

Wait for a line similar to `Keycloak ... started in ...`, then press `Ctrl+C` to
stop following the logs (this does **not** stop the server).

## Step 3 — Log in to the admin console

Open the admin console in your browser:

👉 <http://localhost:8080/admin>

Sign in with the admin credentials from your `.env`:

- **Username:** `admin`
- **Password:** `admin`

You're now in the Keycloak administration UI. In the realm switcher at the top
left, select **techforge** to leave the default `master` realm.

## Step 4 — Explore the pre-configured realm

You don't have to configure the realm by hand — it's imported automatically on
startup from `keycloak/realms/techforge-realm.json`. Take a moment to look
around so you understand what's already set up:

- **Realm roles** (left menu → *Realm roles*): `user` and `admin`.
- **Clients** (left menu → *Clients*): the confidential client `techforge-api`,
  which the NestJS API uses to manage users.
- The `techforge-api` client has a **service account** with the
  `manage-users`, `view-users`, and `query-users` roles, so it's allowed to
  create users through the Admin REST API.

A couple of useful URLs to bookmark:

| What                 | URL                                                                         |
| -------------------- | --------------------------------------------------------------------------- |
| Realm landing page   | <http://localhost:8080/realms/techforge>                                    |
| OpenID configuration | <http://localhost:8080/realms/techforge/.well-known/openid-configuration>   |

The OpenID configuration endpoint lists every URL the API needs (token endpoint,
public keys, and more). Opening it confirms the realm is alive and reachable.

## Step 5 — Get your first access token

Let's prove the service account works by requesting a token the same way the API
does — using the **client credentials** grant:

```bash
curl -s -X POST \
  http://localhost:8080/realms/techforge/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=techforge-api \
  -d client_secret=techforge-api-secret | jq -r .access_token
```

If everything is wired up correctly, you'll get back a long string starting with
`eyJ...` — that's your JWT access token. 🎉

:::note[What's the secret?]
For local development the client secret is `techforge-api-secret`, defined in the
imported realm. In a real deployment you would generate a fresh secret in the
Keycloak console and store it securely.
:::

That token is the key the API uses to call the Admin REST API, for example to
create a new user:

```http
POST http://localhost:8080/admin/realms/techforge/users
Authorization: Bearer <access_token>
```

## Step 6 — Stopping the stack

When you're done for the day, you can stop the containers:

```bash
docker compose down       # stop the containers, keep the database
docker compose down -v     # stop AND delete the Postgres volume (full reset)
```

Use `down -v` whenever you want a completely clean slate — the realm will be
re-imported the next time you start up.

## Troubleshooting

- **Port 8080 already in use** — change `KEYCLOAK_PORT` in `.env` (e.g. `8081`)
  and run `docker compose up -d` again.
- **The console won't load** — give it more time on first boot, then check
  `docker compose logs keycloak` for errors.
- **Token request returns `invalid_client`** — confirm you're hitting the
  `techforge` realm (not `master`) and that the secret is
  `techforge-api-secret`.
- **Nothing starts** — make sure the Docker daemon is running with
  `docker info`.

## What you've learned

In this chapter you:

- ✅ Understood what a realm, client, service account, and token are.
- ✅ Configured your environment and started Keycloak with Docker Compose.
- ✅ Logged into the admin console and explored the `techforge` realm.
- ✅ Requested your very first access token.

## Next steps

With Keycloak running, you're ready to connect the application to it. The next
chapter will cover wiring the **NestJS API** to this Keycloak instance so it can
register and authenticate real users.

---

*This is Chapter 1 of the TechForge newcomer tutorial. More chapters are on the
way!*
