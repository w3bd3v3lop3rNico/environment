# Architettura e concetti

## Panoramica

```
 Host (WSL / Windows)
 ┌──────────────────────────────────────────────────────────┐
 │  :5432 ──┐                         :3000 ──┐             │
 │          │  rete bridge "dev-network"      │             │
 │   ┌──────▼───────┐   postgres:5432  ┌──────▼───────┐     │
 │   │  postgres    │◄─────────────────│  test-app    │     │
 │   │  16-alpine   │   (DNS interno)  │  node 20     │     │
 │   └──────┬───────┘                  └──────────────┘     │
 │          │                                               │
 │   volume devenv_pgdata  (/var/lib/postgresql/data)       │
 └──────────────────────────────────────────────────────────┘
```

- Il `docker-compose.yaml` di root non definisce servizi: **include** `database/compose.yaml` e `test-app/compose.yaml`. Il risultato è un unico progetto Compose (`devenv`) con una sola rete e un solo ciclo di vita.
- Ogni modulo ha il proprio `.env` (variabili passate al container), mentre il `.env` di root serve per l'**interpolazione** (`${POSTGRES_PORT}`, `${APP_PORT}`) e il nome del progetto.

## DNS interno

Su una rete bridge definita dall'utente, Docker fa girare un DNS embedded (`127.0.0.11`) che risolve il **nome del servizio** nell'IP del container. Per questo la test-app usa `DB_HOST=postgres`:

- `localhost` dentro un container indica il container stesso, non l'host né il DB;
- gli IP dei container cambiano a ogni ricreazione, i nomi no;
- se il container `postgres` è fermo, il nome sparisce dal DNS e il client riceve `ENOTFOUND postgres`. Lo vedi in `/db-check` quando fermi il DB.

La porta `5432` pubblicata sull'host (`ports:`) serve **solo** ai client esterni (HeidiSQL, psql dall'host). Tra container il traffico passa direttamente sulla rete interna, senza bisogno di pubblicare porte.

## Healthcheck e ordine di avvio

| Servizio | Check | Significato |
|---|---|---|
| postgres | `pg_isready -U $POSTGRES_USER -d $POSTGRES_DB` | il server accetta connessioni |
| test-app | `wget -qO- http://127.0.0.1:3000/health` | il processo HTTP risponde |

`depends_on: postgres: condition: service_healthy` fa partire la test-app solo quando il DB è **pronto**, non semplicemente avviato. Un container Postgres risulta "running" qualche secondo prima di accettare connessioni, soprattutto al primo avvio, quando esegue gli script di init.

`docker compose ps` mostra lo stato: `starting` → `healthy` / `unhealthy`.

### Liveness vs readiness

- `/health` (**liveness**) non tocca il DB: dice solo che il processo è vivo. Viene usato dall'healthcheck, così un DB momentaneamente giù non fa marcare l'app come guasta.
- `/db-check` (**readiness**) fa un round-trip reale verso Postgres: risponde 200 con i dettagli oppure 503 con l'errore.

## Connection pooling

La test-app usa `pg.Pool`, non una singola connessione:

- `PG_POOL_MAX`: numero massimo di connessioni aperte contemporaneamente;
- `PG_IDLE_TIMEOUT_MS`: dopo quanto tempo una connessione inattiva viene chiusa;
- `PG_CONNECT_TIMEOUT_MS`: quanto attendere una nuova connessione prima di fallire.

Se il DB si riavvia, il pool scarta le connessioni morte (`pool.on('error')` le logga senza far crashare il processo) e alla richiesta successiva ne apre di nuove. `/db-check` riporta `total` / `idle` / `waiting` del pool.

## Persistenza

I dati stanno nel volume nominato `devenv_pgdata`. Il nome viene da `COMPOSE_PROJECT_NAME` + `pgdata`.

- `stop`, `restart` e `down` **non** toccano il volume;
- `down -v` lo elimina;
- gli script in `database/init/` vengono eseguiti **solo** se il volume è vuoto.

## Configurazione della test-app

All'avvio la configurazione viene letta e validata: se manca una variabile il processo termina subito con un log `fatal`, invece di fallire più tardi a runtime. La connection string si costruisce da `DB_*`, oppure si può passare direttamente `DATABASE_URL`. Nei log compaiono host, porta e database, mai la password.

## Riusare il template per un'app reale

1. Copia `test-app/` in una nuova cartella (es. `api/`), poi rinomina il servizio in `compose.yaml` e la porta host.
2. Aggiungi `- path: api/compose.yaml` all'`include` del `docker-compose.yaml` di root.
3. Mantieni `depends_on: postgres: condition: service_healthy` e la rete `dev-network`.
4. Opzionale: crea un database o un utente dedicato aggiungendo uno script in `database/init/` (es. `02-api.sql`), ricordando che gira solo con il volume vuoto.

Nota: un modulo che dipende da `postgres` va avviato dalla root, perché `depends_on` richiede che il servizio sia definito nello stesso progetto.
