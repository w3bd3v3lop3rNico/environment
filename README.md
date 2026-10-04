# environment

Ambiente di sviluppo locale basato su Docker Compose: **PostgreSQL 16** + una **test-app** Node 20/TypeScript che valida la connessione al database. Pensato come template da cui partire per app reali.

```
.
├── docker-compose.yaml     # orchestrazione (include dei moduli)
├── .env.example            # nome progetto + porte host
├── database/
│   ├── compose.yaml        # postgres:16-alpine, volume, healthcheck
│   ├── .env.example        # credenziali DB
│   └── init/01-schema.sql  # eseguito solo al primo avvio (volume vuoto)
├── test-app/
│   ├── compose.yaml        # build, depends_on healthy, healthcheck
│   ├── Dockerfile          # multi-stage, utente non-root
│   ├── .env.example        # connessione DB + pool
│   └── src/index.ts        # /health, /db-check
└── docs/setup.md           # architettura e concetti
```

## Requisiti

- Docker Engine con Compose v2.20+ (serve `include:`)
- Node **non** è necessario sull'host: build ed esecuzione avvengono nei container

## Setup

```bash
cp .env.example .env
cp database/.env.example database/.env
cp test-app/.env.example test-app/.env
# cambia POSTGRES_PASSWORD in database/.env e DB_PASSWORD in test-app/.env (devono coincidere)

docker compose up -d --build
docker compose ps          # entrambi i servizi devono risultare (healthy)
```

## Verifica

```bash
curl localhost:3000/health     # liveness: {"status":"ok",...}
curl localhost:3000/db-check   # query reale sul DB: versione, ora, righe di healthcheck_log, stato pool
```

## Comandi utili

| Comando | Effetto |
|---|---|
| `docker compose up -d --build` | avvia / ricostruisce tutto |
| `docker compose ps` | stato e healthcheck |
| `docker compose logs -f test-app` | log JSON dell'app |
| `docker compose exec postgres psql -U dev -d devdb` | shell SQL |
| `docker compose stop` / `start` | ferma / riavvia senza rimuovere nulla |
| `docker compose down` | rimuove container e rete, **i dati restano** nel volume |
| `docker compose down -v` | rimuove anche il volume: **cancella tutti i dati** |

Esegui i comandi sempre dalla **root del repo** (vedi troubleshooting).

## Connessione da un client GUI (HeidiSQL, DBeaver, …)

| Campo | Valore |
|---|---|
| Host | `127.0.0.1` |
| Porta | `POSTGRES_PORT` (default `5432`) |
| Utente / password | quelli di `database/.env` |
| Database | `devdb` (o quello che vuoi aprire) |

**HeidiSQL**: in modalità PostgreSQL l'albero a sinistra mostra gli *schema* di un solo database, non l'elenco dei database. Indica il database nel campo **Database** della sessione. Se non lo fai, HeidiSQL si collega al DB di default e quelli che crei sembrano "spariti". Per vederne altri apri una sessione per ciascuno, oppure elencali separati da `;`.

## Troubleshooting

**`port is already allocated` / `address already in use`**
La porta è occupata (spesso da un PostgreSQL installato su Windows/host). Cambia `POSTGRES_PORT` o `APP_PORT` nel `.env` di root. Attenzione: con un Postgres nativo sulla 5432, un client GUI su `localhost:5432` potrebbe collegarsi a quello e non al container.

**I dati "spariscono" dopo un riavvio**
I dati vivono nel volume `devenv_pgdata` e sopravvivono a `stop`, `restart` e `down`. Si perdono solo con `down -v` o rimuovendo il volume. Controlla anche di lanciare Compose dalla root: da `database/` il nome del progetto diventa `database` e Compose usa un **altro volume** (`database_pgdata`), vuoto. Per verificare: `docker volume ls`.

**Le modifiche a `database/init/*.sql` non hanno effetto**
Gli script di init girano solo quando il volume è vuoto. Per rieseguirli (cancella i dati): `docker compose down -v && docker compose up -d`.

**`/db-check` risponde 503 `ENOTFOUND postgres`**
Il container `postgres` è fermo o non sta sulla stessa rete: `docker compose ps`, poi `docker compose start postgres`. L'app si riconnette da sola.

**`/db-check` risponde 503 `password authentication failed`**
Le credenziali di `test-app/.env` non coincidono con `database/.env`. Se hai cambiato la password *dopo* il primo avvio, Postgres continua a usare quella vecchia, salvata nel volume. In quel caso aggiornala con `ALTER USER dev PASSWORD '...'` oppure ricrea il volume.

**La test-app non parte e nei log compare `invalid configuration`**
Manca una variabile in `test-app/.env`: il messaggio dice quale.

Approfondimenti su architettura, DNS interno e healthcheck: [docs/setup.md](docs/setup.md).
