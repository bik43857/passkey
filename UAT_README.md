# UAT environment

A third deployment target alongside local dev (`docker-compose.yml`) and production
(`docker-compose.prod.yml`, Phase 9). Same architecture — Browser → HTTPS → Nginx →
Spring Boot → PostgreSQL — on its own domain, TLS cert, database, and Docker volumes, so
it can run side by side with prod on the same host without either interfering with the
other.

## Why UAT still needs HTTPS

WebAuthn/passkeys require a secure context on any origin other than `localhost` — there's
no "skip TLS because it's just a test environment" option here. Point a real subdomain
(e.g. `uat.example.com`) at this server with its own DNS A record before starting.

## What's different from prod

| | prod | uat |
|---|---|---|
| Spring profile | `prod` | `uat` |
| Cookie `SameSite` | `Strict` | `Lax` (tighten to `Strict` if your test flows don't need it) |
| Login/register rate limit | 5 / 3 per minute | 20 / 20 per minute — real limits are easy for a tester re-running the same flow to trip |
| `com.example.auth` logging | INFO | DEBUG |
| Actuator endpoints | `health` only | `health`, `info` |
| DB name / volumes / secrets | `authdb`, `auth_pg_data`, `.env` | `authdb_uat`, `*_uat`, `.env.uat` |

WebAuthn/security packages are still never set to DEBUG, in either environment — that risks
logging challenge values or clientData JSON into log aggregators.

## Bringing it up

```
cp .env.uat.example .env.uat
# edit .env.uat: DOMAIN_NAME (DNS must already point here), DATABASE_PASSWORD,
# SESSION_SECRET (openssl rand -base64 48, DIFFERENT from prod's)

docker compose -f docker-compose.uat.yml --env-file .env.uat up -d --build postgres backend frontend

# One-time certificate issuance (Nginx must already be serving the ACME
# http-01 challenge path from the step above before this will succeed):
docker compose -f docker-compose.uat.yml --env-file .env.uat run --rm certbot certbot certonly \
  --webroot -w /var/www/certbot -d "$DOMAIN_NAME" --email "$CERTBOT_EMAIL" --agree-tos --no-eff-email

docker compose -f docker-compose.uat.yml --env-file .env.uat up -d certbot
```

Same flow as Phase 9's production walkthrough — see `PHASE9_README.md` Section 18 for the
full explanation of each step (renewal loop, why Nginx has no direct backend port, etc.);
only the compose file and env file change.

## Tearing down / resetting UAT data

```
docker compose -f docker-compose.uat.yml --env-file .env.uat down -v
```

`-v` also drops `auth_pg_data_uat`, so this is the "give testers a clean slate" command —
safe to run since it only touches the `*_uat` volumes, never prod's.
