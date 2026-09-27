# smpp_client

High-throughput SMS gateway toolkit built on [Kannel](https://www.kannel.org/) with Redis-backed message store, DLR tracking, and async DLR pipeline to MySQL.

## Architecture

```
Your App
  │  POST /cgi-bin/sendsms?to=...&dlr-url=...&dlr-mask=31
  ▼
smsbox (port 13013)          ← sets msg.dlr_url, msg.id (UUID)
  │
  ▼
bearerbox                    ← Redis DB1: crash-recovery store
  │  submit_sm (SMPP)
  ▼
SMSC                         ← returns foreign_id
  │  deliver_sm (DLR receipt)
  ▼
bearerbox                    ← Redis DB2: DLR table (UUID↔foreign_id)
  │  fires dlr-url HTTP GET
  ▼
dlr_receiver (port 8080)     ← asyncio queue → Redis pipeline XADD
  │
  ▼
Redis Stream dlr_stream      ← burst buffer
  │
  ▼
dlr_writer                   ← batch INSERT 500/commit → MySQL dlr_report
```

## Components

| File | Role |
|---|---|
| `dlr_receiver.py` | Async aiohttp DLR callback server. Batches Redis XADDs via pipeline (18k+/s capacity). |
| `dlr_writer.py` | Async Redis Stream → MySQL batch writer. BATCH_SIZE=500, ON DUPLICATE KEY UPDATE. |
| `send_mts.py` | Parallel async sendsms HTTP sender for load testing. Round-robins across multiple smsbox ports. |
| `schema.sql` | MySQL schema: `mt_log`, `dlr_report`, `v_delivery_hourly`, `v_pending`. |
| `kannel/full_stack.conf` | Bearerbox config: Redis store (DB1), Redis DLR (DB2), SMPP TX+RX, smsbox, sendsms-user. |
| `scripts/tps_bench.sh` | End-to-end TPS benchmark. Measures send TPS, store peak, DLR callback rate. |
| `scripts/sustained_bench.sh` | Long-run stability test. Samples DLR TPS every 5s, reports min/avg/peak. |
| `scripts/run_load_test.sh` | Simple fakesmsc MO injector load test (no sendsms HTTP). |

## Requirements

### System
- macOS or Linux
- Kannel bearerbox + smsbox (built from source — see Build section)
- Redis 6+ (brew install redis)
- MySQL 8+ (for dlr_writer)

### Python
```
pip install -r requirements.txt
```

### Kannel build (macOS Apple Silicon)
```bash
./configure --with-redis --with-redis-dir=/opt/homebrew/Cellar/hiredis/1.4.1 \
  CPPFLAGS="-I/opt/homebrew/opt/libxml2/include/libxml2" \
  LDFLAGS="-L/opt/homebrew/opt/libxml2/lib -L/opt/homebrew/opt/openssl@3/lib"
make -j4
```

## Ports

| Service | Port |
|---|---|
| bearerbox admin | 13000 |
| bearerbox smsbox | 13001 |
| smsbox sendsms | 13013 |
| SMPP (drive_smpp) | 2345 |
| DLR receiver | 8080 |
| Redis | 6379 |

## Redis Layout

| DB | Key | Contents |
|---|---|---|
| 1 | `kannel_test_store` (hash) | In-flight MT blobs (crash recovery) |
| 2 | `kannel_dlr:*` (hashes) | DLR entries: UUID↔foreign_id mapping |
| 2 | `dlr_stream` (stream) | DLR callback queue for dlr_writer |

## Quick Start

```bash
# 1. Start Redis
brew services start redis

# 2. Start DLR receiver
python3 dlr_receiver.py &

# 3. Start fake SMSC (for testing)
./test/.libs/drive_smpp -m 10000 -v 0 &

# 4. Start bearerbox
./gw/.libs/bearerbox -v 0 kannel/full_stack.conf &

# 5. Start smsbox
./gw/.libs/smsbox -v 0 kannel/full_stack.conf &

# 6. Run TPS benchmark
bash scripts/tps_bench.sh 10000 100

# 7. Run sustained stability test
bash scripts/sustained_bench.sh 50000 200
```

## Benchmark Results (macOS Apple Silicon M1)

| Metric | Value |
|---|---|
| Send TPS (sendsms HTTP API) | ~2,200–3,100 msg/s |
| DLR callback TPS (sustained) | ~900–1,500/s |
| Peak 5s-window DLR TPS | 1,542/s |
| Min 5s-window DLR TPS | 867/s |
| Data loss (50,000 msgs) | 0 |
| Errors | 0 |

500 TPS target is **comfortably sustainable** — actual sustained rate is 900–1,500/s.

## Key Config Knobs

```ini
# bearerbox core group
smsbox-max-pending = 500       # flow-control window: bearerbox→smsbox pipe depth

# smsbox group
max-pending-requests = 512     # concurrent outgoing DLR HTTP fires
http-timeout = 10              # fail fast on dead dlr_receiver (default 30s)
http-request-retry = 0         # no retry queue overhead

# smsc group
max-pending-submits = 100      # SMPP window size
```

## DLR Mask Values

| Value | Meaning |
|---|---|
| 1 | Delivered to handset |
| 2 | Failed |
| 4 | Buffered at SMSC |
| 8 | SMSC accepted |
| 16 | SMSC rejected |
| 31 | All events |

## License

MIT
