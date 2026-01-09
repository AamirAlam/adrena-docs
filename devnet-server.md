
# Screen Management Cheat Sheet

A complete guide to using `screen` for managing services on a Linux server.

---

## 1️⃣ Start a new screen

```bash
screen -S <screen_name>


screen -S db           # Database containers
screen -S api          # API service
screen -S processor    # Processor service
screen -S ingestor     # Ingestor service
screen -S enricher     # Enricher service
screen -S cron         # Cron jobs


# DB (Docker)
docker compose -f ~/services/db/docker-compose.db.yml up

# API
cd ~/services/api
pnpm start

# Processor
cd ~/services/processor
pnpm start
```
## Detach from a screen (keep it running)
```Ctrl + A, then D```

## List all active screens
```screen -ls```


## Reattach to a screen
```
Reattach by name
screen -r db
screen -r api

# Force attach if already attached elsewhere
screen -d -r processor
```
## Scroll inside a screen (scrollback mode)

```
Ctrl + A, then [
```

## Rename a screen (optional)
```
Ctrl + A, then A
```

## Kill/terminate a screen
```
Option A: from inside the screen
exit
```

## Option B: from outside
```
screen -S <screen_name> -X quit

Example:
screen -S db -X quit
screen -S api -X quit

```

## Kill all screens (careful!)

```
screen -ls | awk '/Detached/{print $1}' | xargs -n 1 screen -X -S quit
```

## Monitor multiple screens

```
List screens:

screen -ls

Reattach to any:

screen -r <screen_name>


Docker logs:
docker compose -f ~/services/db/docker-compose.db.yml logs -f

Individual container logs:

docker logs -f adrena-source-db
docker logs -f adrena-processor-db

```

###  initialize the grpc submodule
```
# Navigate to adrena-data directory
cd ~/services/enricher/adrena-data

# 1. Check if git submodule is properly initialized
git submodule status

# 2. If grpc is not showing, manually add the submodule
git submodule add https://github.com/rpcpool/yellowstone-grpc grpc

# 3. Update and initialize the submodule
git submodule update --init --recursive --force

# 4. Check if grpc directory exists now
ls -la

# 5. Navigate to the grpc client directory
cd grpc/yellowstone-grpc-client-nodejs

# 6. Install dependencies
npm install

# 7. Build the grpc client
npm run build

# 8. Go to enricher and build
cd ../../enricher
npm install
npm run build
```


