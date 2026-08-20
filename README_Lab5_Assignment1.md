# Lab 5 — Assignment 1: RDS PostgreSQL Deployment & Connection

## Overview
This assignment migrates the Lab 4 Mealie application's database from a local, on-instance PostgreSQL installation to a managed **Amazon RDS PostgreSQL** instance, preserving all existing data and demonstrating full CRUD operations against the new database.

## Architecture

```
Client → Caddy (EC2, :80) → Mealie container (EC2, 127.0.0.1:9000) → Amazon RDS PostgreSQL (private subnet, :5432)
```

The EC2 instance (`mealie-lab4`, i-0074ba1ec324ef939) continues to run the same Mealie v3.22.0 container and Caddy reverse proxy from Lab 4. Only the database layer changed — PostgreSQL now runs on Amazon RDS instead of natively on the EC2 host.

## RDS Configuration

| Setting | Value |
|---|---|
| Engine | PostgreSQL 16 |
| Instance identifier | mealie-db |
| Instance class | db.t3.micro (Free Tier) |
| Storage | 20 GiB (Free Tier default) |
| Public access | **No** — private, reachable only from within the VPC |
| Database name | mealie |
| Master username | mealie |
| Endpoint | mealie-db.c8b80go427hx.us-east-1.rds.amazonaws.com |
| Port | 5432 |

## Security

A dedicated security group, **mealie-rds-sg**, was created for RDS with a single inbound rule:

| Type | Port | Source |
|---|---|---|
| PostgreSQL | 5432 | EC2 security group `sg-010134994e59ab4eb` only |

No `0.0.0.0/0` rule exists on the RDS security group — the database is unreachable from the public internet and only accessible from the Lab 4 EC2 instance itself, satisfying the mandatory security requirement.

## Data Migration

The existing Lab 4 dataset was carried over rather than starting from an empty database:

```bash
# Dump from local PostgreSQL (Lab 4)
pg_dump -U mealie -h 127.0.0.1 mealie > mealie-migrate.sql

# Restore into RDS
psql -h mealie-db.c8b80go427hx.us-east-1.rds.amazonaws.com -U mealie -d mealie -f mealie-migrate.sql
```

Verified post-migration via row counts:
- `recipes`: 3 rows (matches Lab 4 data)
- `users`: 1 row (matches Lab 4 data)

## Application Reconnection

Mealie's environment file (`/etc/mealie/mealie.env`) was updated to point at the RDS endpoint, and the container restarted:

```
DB_ENGINE=postgres
POSTGRES_SERVER=mealie-db.c8b80go427hx.us-east-1.rds.amazonaws.com
POSTGRES_PORT=5432
POSTGRES_DB=mealie
POSTGRES_USER=mealie
POSTGRES_PASSWORD=<redacted>
```

```bash
sudo docker restart mealie
curl -s http://127.0.0.1:9000/api/app/about
# {"production":true,"version":"v3.22.0", ...}
```

## CRUD Demonstration

All four operations were demonstrated live through the running EC2 application, writing directly to RDS:

- **Create** — added a new test recipe
- **Read** — opened an existing migrated recipe (Chicken Parmesan)
- **Update** — edited the test recipe's title/ingredients
- **Delete** — removed the test recipe

## Access

- **EC2 App URL**: http://\<current-public-ip\> (IP auto-updates on instance restart via a systemd automation script; check EC2 console for current value)
- **RDS Endpoint**: mealie-db.c8b80go427hx.us-east-1.rds.amazonaws.com:5432 (private, not publicly reachable)
