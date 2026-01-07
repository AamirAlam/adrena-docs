# Adrena Data - Local Development Setup

## Architecture Overview

The Adrena data pipeline consists of 5 services that work together:

```
Solana Blockchain → Ingestor → Enricher → Processor → API
                                              ↓
                                            Cron Jobs
```

### Services

1. **Ingestor** - Streams Solana transactions via Yellowstone gRPC
2. **Enricher** - Parses and enriches raw transactions
3. **Processor** - Processes enriched transactions into structured data
4. **Cron** - Scheduled jobs for aggregations and calculations
5. **API** - REST API to query processed data

### Databases

- **source_db** (port 5432) - Raw transactions
- **processor_one** (port 5433) - Processed data (primary)
- **processor_two** (port 5434) - Processed data (secondary)

---

## Prerequisites

- Node.js 18+
- PostgreSQL 14+
- Docker & Docker Compose (recommended)
- Solana RPC endpoint
- Yellowstone gRPC endpoint (for ingestor)

---

## Step 1: Database Setup

### Option A: Docker Compose (Recommended)

Create `docker-compose.yml` in the root:

```yaml
version: '3.8'

services:
  source_db:
    image: postgres:14
    container_name: adrena-source-db
    environment:
      POSTGRES_USER: adrena
      POSTGRES_PASSWORD: adrena_dev
      POSTGRES_DB: transaction_db
    ports:
      - "5432:5432"
    volumes:
      - source_data:/var/lib/postgresql/data

  processor_db:
    image: postgres:14
    container_name: adrena-processor-db
    environment:
      POSTGRES_USER: adrena
      POSTGRES_PASSWORD: adrena_dev
      POSTGRES_DB: processor_one
    ports:
      - "5433:5432"
    volumes:
      - processor_data:/var/lib/postgresql/data

  processor_db_two:
    image: postgres:14
    container_name: adrena-processor-db-two
    environment:
      POSTGRES_USER: adrena
      POSTGRES_PASSWORD: adrena_dev
      POSTGRES_DB: processor_two
    ports:
      - "5434:5432"
    volumes:
      - processor_two_data:/var/lib/postgresql/data

volumes:
  source_data:
  processor_data:
  processor_two_data:
```

Start databases:
```bash
docker-compose up -d
```

### Option B: Local PostgreSQL

Create three databases manually:
```bash
createdb transaction_db
createdb processor_one
createdb processor_two
```

---

## Step 2: Environment Configuration

### API Service

Create `api/.env`:
```bash
# Server
PORT=3000
HOST=localhost

# Circuit Breaker
CIRCUIT_BREAKER_REQUESTS=1000
CIRCUIT_BREAKER_TIMEOUT_SECONDS=30

# Source Database
DB_USER=adrena
DB_PASSWORD=adrena_dev
DB_NAME=transaction_db
DB_HOST_EXTERNAL=localhost
DB_HOST_INTERNAL=localhost

# Processor Database (choose 1 or 2)
PROCESSOR_TARGET=1

# Processor One
DB_USER_PROCESSOR=adrena
DB_PASSWORD_PROCESSOR=adrena_dev
DB_NAME_PROCESSOR=processor_one
DB_HOST_EXTERNAL_PROCESSOR=localhost
DB_HOST_INTERNAL_PROCESSOR=localhost

# Processor Two
DB_USER_PROCESSOR_TWO=adrena
DB_PASSWORD_PROCESSOR_TWO=adrena_dev
DB_NAME_PROCESSOR_TWO=processor_two
DB_HOST_EXTERNAL_PROCESSOR_TWO=localhost
DB_HOST_INTERNAL_PROCESSOR_TWO=localhost
```

### Ingestor Service

Create `ingestor/.env`:
```bash
# Database
DB_USER=adrena
DB_PASSWORD=adrena_dev
DB_NAME=transaction_db
DB_HOST=localhost
DB_PORT=5432

# Solana
PROGRAM_ID=13gDzEXCdocbj8iAiqrScGo47NiSuYENGsRqi3SEAwet
```

### Enricher Service

Create `enricher/src/.env`:
```bash
# Database
DB_USER=adrena
DB_PASSWORD=adrena_dev
DB_NAME=transaction_db
DB_HOST=localhost
DB_PORT=5432
```

### Processor Service

Create `processor/.env`:
```bash
# Source Database
DB_USER=adrena
DB_PASSWORD=adrena_dev
DB_NAME=transaction_db
DB_HOST=localhost
DB_PORT=5432

# Processor Database
DB_USER_PROCESSOR=adrena
DB_PASSWORD_PROCESSOR=adrena_dev
DB_NAME_PROCESSOR=processor_one
DB_HOST_PROCESSOR=localhost
DB_PORT_PROCESSOR=5433
```

### Cron Service

Copy and configure:
```bash
cp cron/.env-example cron/.env
```

Edit `cron/.env` with your database credentials and Solana endpoint.

---

## Step 3: Install Dependencies

Install dependencies for each service:

```bash
# API
cd api && npm install

# Ingestor
cd ../ingestor && npm install

# Enricher
cd ../enricher && npm install

# Processor
cd ../processor && npm install

# Cron
cd ../cron && npm install
```

---

## Step 4: Initialize Database Schemas

The schemas are automatically created when services start, but you may need to run migrations or initialization scripts if they exist.

---

## Step 5: Running Services

### Start Services in Order

**Terminal 1 - Ingestor** (streams transactions):
```bash
cd ingestor
npm run start -- subscribe \
  --endpoint <YOUR_YELLOWSTONE_GRPC_ENDPOINT> \
  --x-token <YOUR_TOKEN> \
  --commitment confirmed \
  --cluster mainnet \
  --transactions \
  --transactions-account-include 13gDzEXCdocbj8iAiqrScGo47NiSuYENGsRqi3SEAwet
```

**Terminal 2 - Enricher** (parses transactions):
```bash
cd enricher
npm run start
```

**Terminal 3 - Processor** (processes enriched data):
```bash
cd processor
npm run start
```

**Terminal 4 - API** (serves data):
```bash
cd api
npm run start
```

**Terminal 5 - Cron** (optional, for specific jobs):
```bash
cd cron
npm run start -- -f <function_name>

# Examples:
npm run start -- -f poolSnapshotInfo
npm run start -- -f hourlyStats
npm run start -- -f dailyStats
```

---

## Step 6: Verify Setup

### Check API
```bash
curl http://localhost:3000/swagger
```

### Check Database Connections
```bash
# Source DB
psql -h localhost -U adrena -d transaction_db

# Processor DB
psql -h localhost -p 5433 -U adrena -d processor_one
```

### View Tables
```sql
\dt
```

---

## Development Workflow

### For API Development

1. Make changes in `api/src/`
2. Rebuild: `npm run build`
3. Restart: `npm run start`
4. Test endpoints via Swagger UI or curl

### For New Features

1. **Add new route**: Create file in `api/src/routes/`
2. **Register route**: Add to `api/src/index.ts`
3. **Add database queries**: Use `app.pg.processor` or `app.pg.source`
4. **Test**: Use Swagger or write tests

### For Processor Changes

1. Modify logic in `processor/src/`
2. Rebuild: `npm run build`
3. Restart processor
4. Monitor logs for errors

### For Cron Jobs

1. Create new process in `cron/process/`
2. Add function to `cron/main.ts`
3. Run: `npm run start -- -f yourFunctionName`

---

## Useful Commands

### Build All Services
```bash
for dir in api ingestor enricher processor cron; do
  cd $dir && npm run build && cd ..
done
```

### View Logs
```bash
# Docker logs
docker-compose logs -f source_db
docker-compose logs -f processor_db

# Service logs (if running in background)
tail -f api/logs/*.log
```

### Reset Databases
```bash
docker-compose down -v
docker-compose up -d
```

---

## Troubleshooting

### Port Already in Use
```bash
# Find process
lsof -i :3000

# Kill process
kill -9 <PID>
```

### Database Connection Issues
- Verify credentials in `.env` files
- Check PostgreSQL is running: `docker-compose ps`
- Test connection: `psql -h localhost -U adrena -d transaction_db`

### Missing Dependencies
```bash
cd <service> && npm install
```

### TypeScript Errors
```bash
npm run build
```

---

## Testing Without Full Pipeline

If you don't have access to Yellowstone gRPC, you can:

1. **API Only**: Start just the API with existing database data
2. **Mock Data**: Insert test data directly into databases
3. **Cron Jobs**: Run specific cron jobs to populate data

### Start API Only
```bash
cd api
npm run start
```

The API will work with whatever data exists in the processor database.

---

## Next Steps

- Explore API endpoints at `http://localhost:3000/swagger`
- Review route handlers in `api/src/routes/`
- Check processor logic in `processor/src/`
- Add custom cron jobs in `cron/process/`
- Monitor database tables for data flow

---

## Architecture Notes

### Data Flow
1. **Ingestor** writes raw transactions to `source_db.raw_transactions`
2. **Enricher** reads from `raw_transactions`, parses, writes to `enriched_transactions`
3. **Processor** reads from `enriched_transactions`, processes into structured tables
4. **API** queries processed data from `processor_db`
5. **Cron** runs periodic aggregations and calculations

### Key Tables
- `raw_transactions` - Raw Solana transaction data
- `enriched_transactions` - Parsed transaction data
- `positions` - Trading positions
- `transactions` - Processed transactions
- `pool_snapshot_info` - Pool state snapshots
- `custody_snapshot_info` - Custody state snapshots

### Service Dependencies
- **Enricher** depends on **Ingestor**
- **Processor** depends on **Enricher**
- **API** depends on **Processor** (data must exist)
- **Cron** depends on **Processor** (for aggregations)
