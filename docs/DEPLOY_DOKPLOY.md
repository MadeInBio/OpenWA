# Deploy di OpenWA su Dokploy (server Oracle Cloud ARM)

Guida per pubblicare OpenWA sulla tua istanza **Dokploy** usando il file
[`docker-compose.dokploy.yml`](../docker-compose.dokploy.yml).

Configurazione di riferimento:
- **Server**: Oracle Cloud ARM Ampere A1 (arm64)
- **Database dati**: PostgreSQL (nel compose)
- **Esposizione**: dominio dedicato con HTTPS automatico (Let's Encrypt via Traefik di Dokploy)

> Architettura: Dokploy ha già il suo **Traefik** con SSL. Esponiamo al pubblico solo
> la **dashboard** (nginx), che fa da reverse-proxy interno verso l'API per le rotte
> `/api/` e `/socket.io/`. Un solo dominio serve sia la UI sia l'API.

---

## 1. Prerequisiti

1. **Dokploy** installato e funzionante sul server Oracle.
2. Il repository **`MadeInBio/OpenWA`** su GitHub deve contenere il file
   `docker-compose.dokploy.yml` (vedi passo 2).
3. Un **(sotto)dominio** con record DNS **A** che punta all'**IP pubblico** del server, es:
   ```
   wa.miodominio.com   A   <IP_PUBBLICO_ORACLE>
   ```
   > Su Oracle Cloud ricordati di aprire le porte **80** e **443** sia nella
   > *Security List / Network Security Group* della VCN, sia nel firewall del SO
   > (`iptables`/`firewalld`), altrimenti Let's Encrypt e il traffico HTTPS non passano.

---

## 2. Porta il compose nel repository

Dokploy fa il deploy **clonando il repo da GitHub**, quindi il file
`docker-compose.dokploy.yml` deve essere committato e pushato sul branch che userai
(es. `main`):

```bash
git add docker-compose.dokploy.yml docs/DEPLOY_DOKPLOY.md
git commit -m "chore(deploy): aggiungi compose e guida per Dokploy"
git push origin main
```

---

## 3. Genera i segreti

Ti servono due valori casuali. Esempi:

PowerShell (Windows):
```powershell
# API_MASTER_KEY
[Convert]::ToBase64String((1..48 | ForEach-Object { Get-Random -Max 256 }))
# DATABASE_PASSWORD
[Convert]::ToBase64String((1..24 | ForEach-Object { Get-Random -Max 256 }))
```

Linux/macOS:
```bash
openssl rand -base64 48   # API_MASTER_KEY
openssl rand -base64 24   # DATABASE_PASSWORD
```

Conserva entrambi: `API_MASTER_KEY` è la chiave admin dell'API.

---

## 4. Crea il progetto Compose in Dokploy

1. **Dashboard Dokploy → Projects → Create Project** (es. nome `openwa`).
2. Dentro al progetto: **Create Service → Compose**.
3. **Provider**: *Git* → collega GitHub e seleziona il repo **`MadeInBio/OpenWA`**.
   - **Branch**: `main`
   - **Compose Path**: `./docker-compose.dokploy.yml`
4. **Compose Type**: `Docker Compose`.

---

## 5. Variabili d'ambiente

Nella scheda **Environment** del servizio Compose incolla (sostituendo i valori):

```env
DOMAIN=wa.miodominio.com
API_MASTER_KEY=<la-chiave-generata-al-passo-3>
DATABASE_PASSWORD=<la-password-generata-al-passo-3>
```

Queste variabili vengono interpolate dentro `docker-compose.dokploy.yml`
(`${DOMAIN}`, `${API_MASTER_KEY}`, `${DATABASE_PASSWORD}`).

---

## 6. Dominio / HTTPS

Il routing e il certificato sono già definiti dalle **label Traefik** nel compose,
quindi di norma **non serve** aggiungere nulla nella scheda *Domains* di Dokploy:
ti basta aver impostato `DOMAIN` e aver puntato il DNS.

> Se preferisci gestire il dominio dalla UI *Domains* di Dokploy invece che dalle label:
> rimuovi il blocco `labels:` del servizio `dashboard` nel compose e aggiungi un dominio
> indicando **Service Name** = `dashboard` e **Port** = `80`.
>
> I nomi dei router/middleware Traefik (`openwa`, `openwa-web`, `openwa-https`) sono
> globali: se hai già un altro progetto con quegli stessi nomi, rinominali per evitare conflitti.

---

## 7. Deploy

1. Premi **Deploy**.
2. Il primo deploy **builda da sorgente** (API + dashboard): su ARM Ampere richiede
   qualche minuto perché compila i moduli nativi e prepara Chromium.
3. Segui i **Logs** in Dokploy. Sei a posto quando vedi l'avvio di Nest e l'healthcheck
   dell'API diventa *healthy* (`/api/health`).

Verifica:
```bash
curl https://wa.miodominio.com/api/health
```

---

## 8. Primo avvio: API key e sessione WhatsApp

1. Apri `https://wa.miodominio.com` → la **dashboard**.
2. Autenticati / configura la **API key** usando la `API_MASTER_KEY` impostata.
   (In alternativa puoi creare una API key applicativa dalla dashboard o via endpoint
   protetto dalla master key.)
3. Crea una **sessione**, avviala e scansiona il **QR code** con WhatsApp
   (Dispositivi collegati → Collega un dispositivo).

Swagger/OpenAPI: `https://wa.miodominio.com/api/docs`

---

## 9. Persistenza e backup (IMPORTANTE)

Due volumi Docker tengono lo stato:

| Volume          | Contenuto                                                        |
| --------------- | --------------------------------------------------------------- |
| `openwa-data`   | `main.sqlite` (**API key + audit**), `sessions/` (**auth WhatsApp**), `media/`, `plugins/` |
| `postgres-data` | dati di sessioni/webhook/messaggi (PostgreSQL)                  |

> ⚠️ Se cancelli/perdi `openwa-data` devi **riscansionare il QR** di ogni sessione e
> perdi le API key. Includi entrambi i volumi nei backup. Dokploy offre backup
> programmati per i volumi/DB: configurali su `openwa-data` e sul database.

---

## 10. Note su risorse (ARM Ampere)

- Ogni sessione WhatsApp avvia un'istanza **Chromium** headless: stima **~300-500 MB RAM**
  per sessione. Con la A1 Flex (≥6 GB) gestisci tranquillamente più sessioni.
- Il `Dockerfile` installa Chromium dai pacchetti Debian **arm64** e imposta
  `PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium`: nessuna configurazione extra per ARM.
- `shm_size: 1gb` e `--disable-dev-shm-usage` nel compose prevengono i crash di Chromium
  dovuti a `/dev/shm` troppo piccolo.

---

## 11. Aggiornamenti

Per aggiornare OpenWA: fai `git push` delle modifiche sul branch collegato e premi
**Redeploy** in Dokploy (o abilita l'**auto-deploy** via webhook GitHub).
I volumi persistono tra un deploy e l'altro, quindi sessioni e dati restano intatti.

---

## 12. Cosa è stato escluso rispetto al `docker-compose.yml` originale (e perché)

| Elemento                         | Stato      | Motivo                                                                 |
| -------------------------------- | ---------- | --------------------------------------------------------------------- |
| Servizio **Traefik** interno     | rimosso    | Dokploy ha già il suo Traefik con SSL automatico.                     |
| **Port binding** `127.0.0.1:...` | rimosso    | L'esposizione la gestisce il Traefik di Dokploy via label.            |
| Mount `/var/run/docker.sock`     | rimosso    | Evita di esporre il socket Docker; la sezione "infra" della dashboard è degradata, l'app funziona. |
| Profili `minio` / `redis`        | non inclusi| Storage locale e niente code/cache nel deploy base. Aggiungibili in seguito. |

### (Opzionale) Riabilitare Redis per code/BullMQ
Se in futuro vuoi le code BullMQ + Bull Board, aggiungi un servizio `redis:7-alpine`
sulla rete `openwa-internal` e imposta sull'API:
`REDIS_ENABLED=true`, `REDIS_HOST=redis`, `REDIS_PORT=6379`.
