# Demo: Redis to Valkey Migration with BetterDB

This demo walks through a complete Redis-to-Valkey migration using BetterDB's three-phase workflow: **Analysis**, **Execution**, and **Validation**.

## Prerequisites

- Docker and Docker Compose installed
- Ports 6379, 6399, and 3001 available

## Architecture

```
+------------------+          +------------------+
|  Redis 7.4       |  ------> |  Valkey 8.1      |
|  (source)        |  migrate |  (target)        |
|  localhost:6379   |          |  localhost:6399   |
+------------------+          +------------------+
         |
         |  monitors
         v
+------------------+
|  BetterDB        |
|  localhost:3001   |
+------------------+
```

## Step 1: Start the Environment

```bash
docker compose up -d
```

This starts three containers:

| Container | Image | Port | Role |
|-----------|-------|------|------|
| redis-source | redis:7.4 | 6379 | Source instance with data |
| valkey-target | valkey/valkey:8.1 | 6399 | Empty target instance |
| betterdb | betterdb/monitor | 3001 | Migration + monitoring tool |

Verify everything is running:

```bash
docker compose ps
```

## Step 2: Load Sample Data

Run the data loader script to populate Redis with a realistic mix of data types:

```bash
./load-data.sh
```

This creates:

| Key Pattern | Type | Count | Description |
|-------------|------|-------|-------------|
| `user:{id}:name` | string | 5 | User display names |
| `user:{id}:profile` | hash | 5 | User profiles with multiple fields |
| `user:{id}:notifications` | list | 5 | Notification queues (10 items each) |
| `tags:{category}` | set | 3 | Tag collections |
| `leaderboard:daily` | sorted set | 20 | Player scores |
| `leaderboard:weekly` | sorted set | 20 | Player scores |
| `session:{token}` | string (with TTL) | 10 | Active sessions expiring in 1 hour |
| `cache:page:{slug}` | string (with TTL) | 5 | Cached HTML pages expiring in 10 min |
| `config:app` | hash | 1 | Application configuration |
| `events:stream` | stream | 50 | Event log entries |

**Total: ~124 keys** across 6 data types, with a mix of persistent and expiring keys.

Verify the data loaded:

```bash
docker exec redis-source redis-cli DBSIZE
# Expected: (integer) 124

docker exec redis-source redis-cli INFO keyspace

# output
27.0.0.1:6379> dbsize
(integer) 37


127.0.0.1:6379> INFO keyspace
# Keyspace
db0:keys=37,expires=15,avg_ttl=2327002,subexpiry=0
```

## Step 3: Open BetterDB

Open your browser to:

```
http://localhost:3001
```

You will see the BetterDB dashboard showing live metrics from the Redis source instance: memory usage, connected clients, ops/sec, and more.

## Step 4: Add the Valkey Target Connection

BetterDB starts connected to Redis (port 6379). You need to add the Valkey target as a second connection.

1. Click the **connection selector** in the top navigation bar
2. Click the **"+"** button to add a new connection
3. Enter the following details:
   - **Name:** Valkey Target
   - **Host:** valkey-target (the Docker network hostname)
   - **Port:** 6379 (internal container port; Docker maps 6399 externally)
4. Click **Save**

Both connections are now registered in BetterDB.

> **Note:** If you are running BetterDB outside Docker (e.g., via `npx`), use `localhost` as the host and `6399` as the port for the Valkey target.

<img width="1744" height="728" alt="image" src="https://github.com/user-attachments/assets/ff0fc69e-715b-45fb-92c6-169579caa923" />

<img width="1379" height="198" alt="image" src="https://github.com/user-attachments/assets/bb14915a-8106-4038-a89c-a827433d0532" />


## Step 5: Run the Migration Analysis

1. Navigate to the **Migration** section in BetterDB
2. Select the **Redis source** (port 6379) as the source
3. Select the **Valkey target** (port 6379 / 6399) as the target
4. Leave the sample size at the default (or increase it)
5. Click **"Analyze"**

<img width="1339" height="652" alt="image" src="https://github.com/user-attachments/assets/d73d2d3d-b8d6-4d9b-8439-d41443118333" />

<img width="1919" height="914" alt="image" src="https://github.com/user-attachments/assets/488f260c-1acd-43cd-a99d-7c3cf4fb4398" />


### What to Expect

BetterDB scans the source keyspace and produces a compatibility report:

**Compatibility verdict:** Green (no blocking issues for Redis-to-Valkey)

**Data type breakdown:**
- Strings: count + estimated memory
- Hashes: count + estimated memory
- Lists: count + estimated memory
- Sets: count + estimated memory
- Sorted sets: count + estimated memory
- Streams: count + estimated memory

**TTL distribution:**
- Keys with no expiry (config, profiles, leaderboards)
- Keys expiring <1h (sessions)
- Keys expiring <24h (cached pages)

**No data has been written to the target at this point.** The analysis is read-only.

## Step 6: Execute the Migration

1. After reviewing the analysis, click **"Execute"**
2. Select **Command mode** (recommended for cross-protocol Redis-to-Valkey migrations)
3. Click **Start**

<img width="1916" height="855" alt="image" src="https://github.com/user-attachments/assets/fc3ad6c8-6509-4331-a837-9cbd7ffd73c8" />

<img width="693" height="391" alt="image" src="https://github.com/user-attachments/assets/91cd6037-a822-4e52-afc6-7baa8d22bc67" />

<img width="1626" height="478" alt="image" src="https://github.com/user-attachments/assets/2b182e54-7cfd-4dd4-a5fa-f8dc2cf6735b" />



### What Happens During Execution

- BetterDB reads each key using the optimal command for its data type
- Compound types (hashes, lists, sets, sorted sets, streams) are written to a temporary staging key, then atomically renamed
- TTLs are preserved using `SET PX` for strings and Lua `RENAME + PEXPIRE` for compound types
- Real-time progress shows: transferred count, skipped count, estimated completion
- A live log viewer shows each operation

### Expected Result

```
Keys transferred: 124
Keys skipped: 0
Errors: 0
```

## Step 7: Validate the Migration

1. Click **"Validate"**
2. BetterDB runs three checks:

| Check | What It Does |
|-------|--------------|
| Key count | `DBSIZE` on both sides, flags >1% discrepancy |
| Sample spot-check | Random keys compared: type + binary-safe value match |
| Baseline comparison | Target's ops/sec, memory, fragmentation vs. source pre-migration |

<img width="1429" height="685" alt="image" src="https://github.com/user-attachments/assets/1ec49661-1432-4738-b006-355109a3c81e" />


### Expected Result

**Validation: PASS**

All key counts match. All sampled keys have matching types and values. TTLs preserved.

<img width="1915" height="851" alt="image" src="https://github.com/user-attachments/assets/f7f8a579-d2dd-4d2a-8c24-9d3ea8c441c0" />

## Step 8: Manual Verification (Optional)

Verify the data arrived in Valkey:

```bash
# Check key count
docker exec valkey-target valkey-cli DBSIZE

# Check string data
docker exec valkey-target valkey-cli GET user:1:name

# Check hash data
docker exec valkey-target valkey-cli HGETALL user:1:profile

# Check list data
docker exec valkey-target valkey-cli LRANGE user:1:notifications 0 -1

# Check set data
docker exec valkey-target valkey-cli SMEMBERS tags:devops

# Check sorted set data
docker exec valkey-target valkey-cli ZRANGE leaderboard:daily 0 -1 WITHSCORES

# Check stream data
docker exec valkey-target valkey-cli XLEN events:stream

# Check TTL on a session key
docker exec valkey-target valkey-cli TTL session:token-001
```

## Cleanup

```bash
docker compose down -v
```

This stops and removes all containers and volumes.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port 6379/6399/3001 already in use | Stop existing Redis/Valkey/app on those ports, or modify `docker-compose.yml` |
| BetterDB cannot connect to Redis | Ensure `redis-source` container is healthy: `docker compose ps` |
| Valkey target connection fails | Use `valkey-target` as hostname (Docker networking), not `localhost` |
| Migration analysis shows blocking issues | Review the compatibility report; your setup may differ from this demo |
