# Infrastructure — Shared Development Environment

Centralized Docker Compose providing shared infrastructure for all microservices:
databases (PostgreSQL) and message broker (Apache Kafka).

## Quick Start

```bash
docker compose up -d
```

This starts all infrastructure services. No further configuration needed for local development.

## Port Map

| Service | Container Port | Host Port | Database Name | Used By |
|---|---|---|---|---|
| `kafka` | 9092 | **9092** | — | All services via `spring.kafka.bootstrap-servers` |
| `equipment-db` | 5432 | **5432** | `yaku_equipment` | equipment-service |
| `subscription-db` | 5432 | **5433** | `mydatabase` | subscription-service |
| `iam-db` | 5432 | **5434** | `yaku_iam` | iam-service |
| `notification-db` | 5432 | **5435** | `yaku_notifications` | notification-service |

## Service Connection Details

### PostgreSQL (all databases)
- **Host:** `localhost`
- **Port:** See port map above
- **User:** `root`
- **Password:** `password`
- **Auth:** `md5` (default)

Example connection string for iam-service:
```
jdbc:postgresql://localhost:5434/yaku_iam
```

### Apache Kafka (KRaft mode, no Zookeeper)
- **Bootstrap server:** `localhost:9092`
- **Protocol:** PLAINTEXT (no SASL)
- **Auto-create topics:** enabled (`KAFKA_AUTO_CREATE_TOPICS_ENABLE=true`)
- **Topics created on demand:** yes

## Kafka Topics

Services publish events to these topics:

| Topic | Publisher | Consumers |
|---|---|---|
| `iam.user-registered` | iam-service | equipment-service (future) |

## Kafka Networking Note

Kafka uses `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092` so that clients
connecting from the host machine (`localhost:9092`) can reach the broker by its
container hostname (`yaku-kafka`). If you see errors like:

```
Error connecting to node yaku-kafka:9092
```

The advertised listener is misconfigured. Ensure `KAFKA_ADVERTISED_LISTENERS` matches
the port you use to connect from the host.

## Stopping

```bash
docker compose down
```

To also wipe data volumes:

```bash
docker compose down -v
```

## Individual vs Centralized Infrastructure

Each service has its own `compose.yaml` with isolated Kafka and Postgres for
standalone development/testing. Use **this** centralized compose when testing
multiple services together or running through the API Gateway.

| Scenario | Use |
|---|---|
| Single service development | Service's own `compose.yaml` |
| Full system / gateway testing | This centralized `compose.yaml` |
| equipment + iam integration | This centralized `compose.yaml` |

## Health Checks

Postgres containers use `pg_isready` for health checks. Kafka does not have a
health check defined — use `docker compose logs kafka` to verify startup.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              yaku-network (bridge)                  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  kafka   │  │equipment-│  │   subscription-  │  │
│  │  :9092   │  │   db     │  │       db         │  │
│  └──────────┘  │  :5432   │  │     :5433        │  │
│                └──────────┘  └──────────────────┘  │
│                ┌──────────┐  ┌──────────────────┐  │
│                │  iam-db  │  │ notification-db  │  │
│                │  :5434   │  │     :5435        │  │
│                └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────┘
```