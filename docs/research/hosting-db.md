# Research: Hosting und Datenbank für TanStack Start unter 5 €/Monat

> Ticket: [#3](https://github.com/ExziiL/snack_v5/issues/3) (Map [#1](https://github.com/ExziiL/snack_v5/issues/1))
> Stand der Preise und Limits: **abgerufen am 2026-10-30** von den offiziellen Seiten der Anbieter. Preise ändern sich häufig, vor der Umsetzung noch einmal prüfen. Angaben in US-$ sind Originalpreise. Wechselkurs nicht eingerechnet.

## Frage

Welche Hosting-Ziele unterstützt TanStack Start heute offiziell, welche Datenbanken passen dazu für eine Einzelnutzer-App, was kosten sie (Ziel < ~5 €/Monat, Free Tier bevorzugt), wie gut passen sie zu den Sync-Optionen von TanStack DB, und wie aufwendig sind Secrets bei einem öffentlichen Repo?

## Kurzfassung

- **Offizielle Partner von TanStack Start:** Cloudflare Workers, Netlify und Railway. Vercel, Node.js/Docker und Bun laufen über Nitro, Appwrite Sites hat ebenfalls eine Anleitung. Fly.io wird nicht erwähnt. [1]
- **Empfehlung A (Standard, 0 €):** **Cloudflare Workers Free + Cloudflare D1 (SQLite)**. TanStack DB dazu mit **QueryCollection** über Server Functions. ElectricSQL und PowerSync gehen damit nicht, weil beide Postgres (oder MySQL/MongoDB) brauchen.
- **Empfehlung B (falls echter Sync über ElectricSQL gewünscht ist):** **Cloudflare Workers + Supabase Free (Postgres) + Electric Cloud (PAYG, unter 5 $/Monat erlassen)**. Neon Free passt schlechter, weil eine dauerhaft offene Replikationsverbindung Scale-to-Zero verhindert und das Compute-Kontingent schnell verbraucht ist.
- **Ausweg bei Limits:** Workers Paid kostet 5 $/Monat und hebt das CPU-Limit von 10 ms auf 30 s (Standard) an. Damit bleibt man im Rahmen von ca. 5 €.

## 1. Hosting-Ziele von TanStack Start

Laut dem offiziellen Hosting-Guide [1]:

| Ziel | Status im Guide | Adapter / Setup |
|---|---|---|
| Cloudflare Workers | **Offizieller Partner** | `@cloudflare/vite-plugin` (`cloudflare({ viteEnvironment: { name: 'ssr' } })`), `wrangler.jsonc` mit `main: "@tanstack/react-start/server-entry"`, Deploy mit `wrangler deploy` |
| Netlify | **Offizieller Partner** | `@netlify/vite-plugin-tanstack-start`, Deploy mit `npx netlify deploy` |
| Railway | **Offizieller Partner** | Nitro (`nitro/vite`), dann Repo bei Railway verbinden |
| Vercel | unterstützt | Nitro, dann One-Click-Deploy |
| Node.js / Docker | unterstützt | Nitro, Start mit `node .output/server/index.mjs` (für Docker gibt es nur die Überschrift, keine eigenen Schritte) |
| Bun | unterstützt | Nitro mit `bun`-Preset oder eigenes `server.ts`, React 19 nötig |
| Appwrite Sites | unterstützt | Repo verbinden, Output `./dist` bzw. `./.output` |
| Fly.io | **nicht erwähnt** | ginge nur als generischer Node/Docker-Server |

Auch Cloudflare selbst pflegt einen Guide für TanStack Start auf Workers. Bindings (D1, R2, KV, Secrets) liest man in Server Functions über `import { env } from "cloudflare:workers"`. Rsbuild wird dort nicht unterstützt, und ein vorhandener `nitro()`-Adapter muss entfernt werden [2].

## 2. Hosting: Free Tiers und Kosten

| Anbieter | Free Tier | Günstigster Bezahlplan | Relevante Grenzen für Snack |
|---|---|---|---|
| **Cloudflare Workers** [3][4][5] | 100.000 Requests/Tag, **10 ms CPU pro Request**. Wartezeit auf `fetch()`/DB zählt nicht als CPU. Static-Asset-Requests sind kostenlos und unbegrenzt. | Workers Paid **5 $/Monat** (10 Mio. Requests, 30 Mio. CPU-ms inklusive, bis 5 min CPU/Request, Standard 30 s) | Request-Body bis 100 MB (für Bon-Fotos unkritisch). 50 Subrequests/Request (Free). 128 MB Speicher. |
| **Vercel Hobby** [6][7] | 0 $, nur für **„personal, non-commercial use“**. 1 Mio. Invocations, 4 h Active CPU, 360 GB-h Speicher, 100 GB Transfer pro Monat. Zusätzliche Nutzung ist nicht kaufbar. | Pro 20 $/Monat (außerhalb des Budgets) | **Request-Body max. 4,5 MB**, deshalb Bon-Fotos vorher im Client verkleinern. Max. 300 s Laufzeit. |
| **Netlify Free** [8][9][10] | 300 Credits/Monat, hartes Limit (Compute 10 Credits/GB-h, 2 Credits pro 10.000 Requests, 20 Credits/GB Bandbreite, **15 Credits pro Production-Deploy**) | Personal 9 $/Monat (außerhalb des Budgets) | Sind die Credits aufgebraucht, wird die **Seite bis zum nächsten Abrechnungszyklus pausiert** („Site not available“). 20 Deploys pro Monat verbrauchen bereits alle 300 Credits. |
| **Railway** [11][12][13] | Trial: einmalig 5 $ für 30 Tage. Danach Free mit 1 $ Guthaben/Monat. | Hobby **5 $/Monat inkl. 5 $ Nutzung**, darüber nutzungsbasiert (RAM ca. 10 $/GB-Monat, vCPU ca. 20 $/Monat) | Ein dauerhaft laufender Node-Dienst mit 0,5 GB RAM kostet allein schon ca. 5 $. „Serverless“ schläft nach 5–10 min ohne ausgehenden Traffic, danach gibt es einen Cold Start, eventuell mit 502 beim ersten Request. |
| **Fly.io** [14] | **Kein Free Tier** für neue Organisationen, nur eine Trial (2 h Laufzeit oder 7 Tage) | kleinste Maschine (shared-cpu-1x, 256 MB) 2,19 $/Monat bei Dauerbetrieb | Managed Postgres ab 38 $/Monat. Nicht offiziell im TanStack-Guide. |

**Einordnung:** Für eine Einzelnutzer-App mit wenigen Dutzend Requests am Tag sind nur **Cloudflare** und **Vercel** wirklich dauerhaft kostenlos. Netlify ist kostenlos, solange man wenig deployt, aber das Pausieren bei Credit-Ende ist für ein Werkzeug, das man täglich benutzt, ein Risiko. Railway und Fly kosten real Geld und haben ohne Dauerbetrieb Cold Starts.

**Risiko bei Cloudflare Free:** 10 ms CPU pro Request reichen für SSR einer kleinen App und für das Durchreichen von Daten in der Regel aus, garantiert ist das aber nicht. Netzwerk-Wartezeit (DB, Vision-API) zählt nicht mit [4]. Kritisch wäre CPU-intensive Arbeit im Worker, etwa ein großes Bild serverseitig in Base64 umzuwandeln. Abhilfe: das Foto im Client verkleinern und komprimieren, oder auf Workers Paid (5 $) wechseln.

## 3. Datenbanken: Free Tiers, Pausieren, Kosten

| DB | Typ | Free Tier | Pausieren / Cold Start | Bezahlplan |
|---|---|---|---|---|
| **Cloudflare D1** [3][15][16] | SQLite (serverless, Binding im Worker) | 5 Mio. Rows read/Tag, 100.000 Rows written/Tag, 5 GB gesamt, **500 MB pro DB**, 10 DBs, **Time Travel 7 Tage** | Kein Pausieren. Über dem Limit gibt es Fehler, keine Kosten. | im Workers Paid (5 $) enthalten: 25 Mrd. Reads, 50 Mio. Writes, 5 GB |
| **Neon** [17][18] | Postgres (serverless) | 100 CU-h/Monat pro Projekt, 1 GB Storage pro Projekt, 5 GB Egress, 6 h History | **Scale-to-Zero nach 5 min Inaktivität** (nicht abschaltbar), danach Cold Start. Wenn ein Limit erreicht ist, wird Compute bis zum nächsten Monat suspendiert. **Ein verbundener Logical-Replication-Subscriber (z. B. Electric, PowerSync) verhindert Scale-to-Zero.** | Launch ohne Mindestbetrag: 0,106 $/CU-h, 0,35 $/GB-Monat |
| **Supabase** [19][20] | Postgres (+ Auth, Storage) | 500 MB DB, 5 GB Egress, 2 aktive Projekte | **Pausiert nach 7 Tagen geringer Aktivität**. Danach ist die DB weg, bis man sie manuell im Dashboard wiederherstellt (möglich bis zu 1 Jahr). Ein paar Requests pro Tag reichen als Aktivität. | Pro 25 $/Monat (außerhalb des Budgets) |
| **Turso** [21] | SQLite/libSQL (über HTTP) | 100 DBs, 5 GB, 500 Mio. Rows read/Monat, 10 Mio. Rows written/Monat, 3 GB Syncs, 1 Tag PITR | Auf der Pricing-Seite steht nichts dazu | Developer 4,99 $/Monat |
| Railway Postgres / Fly MPG [11][14] | Postgres | – | – | nutzungsbasiert (Railway) bzw. ab 38 $ (Fly) |

**Rechnung zu Neon + Electric/PowerSync:** Eine Replikationsverbindung, die ständig offen ist, hält die Compute dauerhaft wach [18]. Ein Monat hat ca. 730 h. Schon bei 0,25 CU (kleinste übliche Neon-Compute, **Annahme, auf der Pricing-Seite nicht ausgewiesen**) wären das ca. 182 CU-h, also über den 100 CU-h des Free Tiers. Im Launch-Plan kämen ca. 82 CU-h × 0,106 $ ≈ 8,7 $ dazu. Neon passt also nur mit QueryCollection, nicht mit Dauer-Sync.

**Supabase + Sync:** Die Compute läuft im Free Tier ohnehin dauerhaft (nicht nutzungsbasiert abgerechnet). Ein Sync-Dienst, der regelmäßig zugreift, verursacht daher keine Zusatzkosten. Das 7-Tage-Pausieren ist bei täglicher Nutzung unkritisch. Nach einem Urlaub ohne Nutzung müsste man die DB aber manuell wiederherstellen.

## 4. Kompatibilität mit den Sync-Optionen von TanStack DB

TanStack DB ist aktuell auf der 1.0-RC-Linie und bietet laut Übersicht [22]: QueryCollection, ElectricCollection, PowerSyncCollection, TrailBaseCollection, RxDBCollection sowie lokale Collections (IndexedDB, LocalStorage, LocalOnly). Dazu gibt es Guides zu Offline-Transaktionen (ausstehende Mutationen behalten und später erneut senden) und SQLite-Persistenz.

| Collection | Backend-Anforderung | D1 | Turso | Neon | Supabase |
|---|---|---|---|---|---|
| **QueryCollection** [23] | beliebige API, z. B. TanStack-Start-Server-Functions. `queryFn` liefert den vollen Datenstand, Mutationen über `onInsert`/`onUpdate`/`onDelete` | ✅ | ✅ | ✅ | ✅ |
| **ElectricCollection** [24] | Postgres mit Logical Replication + Electric-Dienst (Cloud oder selbst gehostet), Proxy-Route in TanStack Start. Writes gehen über die eigene API, die eine `txid` zurückgibt (`pg_current_xact_id()` in derselben Transaktion) | ❌ | ❌ | ⚠️ kostet (s. o.) | ✅ |
| **PowerSyncCollection** [22][25] | PowerSync-Dienst mit Postgres/MySQL/MongoDB/SQL Server, kein SQLite-Backend | ❌ | ❌ | ⚠️ kostet (s. o.) | ✅ technisch, aber s. u. |
| TrailBaseCollection [22] | selbst gehosteter TrailBase-Server | ❌ (braucht eigenen Server, also Railway/Fly-Kosten) | | | |
| RxDBCollection [22] | eigene RxDB-Replikation | nicht weiter untersucht | | | |

**Kosten der Sync-Dienste:**
- **Electric Cloud** [26]: PAYG ohne Grundgebühr und ohne Kreditkarte, 1 $ pro 1 Mio. Writes und 0,10 $/GB-Monat Retention, Reads und Egress kostenlos. **Unter 5 $/Monat wird nichts berechnet.** Für eine Einzelnutzer-App also faktisch 0 €. Die Pricing-Seite sagt nichts dazu, welche Postgres-Anbieter offiziell unterstützt werden. Neon und Supabase werden dort nicht ausdrücklich bestätigt.
- **PowerSync Cloud** [25]: Free mit 2 GB Sync/Monat, 500 MB gehostet, 50 Clients. **Freie Projekte werden nach Inaktivität deaktiviert** (die Seite nennt widersprüchlich 1 bzw. 2 Wochen). Pro ab 49 $/Monat, deshalb außerhalb des Budgets. Selbst hosten (Open Edition) bräuchte wieder einen Server.

**Folgerung:** Für den Offline-Fall (Snack unterwegs ohne Netz erfassen) reicht vermutlich **QueryCollection + Offline-Transaktionen/lokale Persistenz** im Client. Ob das genügt, klärt das eigene TanStack-DB-Ticket. Damit bleibt die Wahl der DB frei, und **D1 ist die günstigste und einfachste Option**. Echter Server-Push-Sync (Electric) bringt bei einem einzigen Nutzer mit einem Gerät wenig Mehrwert.

## 5. Secrets und Env-Variablen bei öffentlichem Repo

**TanStack Start** [27]:
- Nur Variablen mit `VITE_`-Prefix landen im Client. **Secrets (z. B. der Vision-API-Key) also nie mit `VITE_` benennen.**
- Server Functions lesen Variablen ohne Prefix über `process.env`. Diese Werte erreichen den Client nicht.
- `process.env` **pro Request** im Handler lesen, nicht auf Modulebene. Auf Edge-Runtimes kann ein Zugriff auf Modulebene `undefined` liefern oder das Secret ins Bundle schreiben. Auf Cloudflare ist `env` aus `cloudflare:workers` der vorgesehene Weg.
- `.env.local` für lokale Werte, nicht in Git. Produktionswerte setzt man beim Hoster. Pflichtvariablen beim Start validieren (z. B. mit Zod).

**Cloudflare** [28]:
- Produktion: `npx wrangler secret put VISION_API_KEY` oder im Dashboard (Variables and Secrets → Typ „Secret“). Gesetzte Secrets sind danach **weder in Wrangler noch im Dashboard sichtbar**.
- Lokal: `.dev.vars` oder `.env` neben `wrangler.jsonc`. **`.dev.vars*` und `.env*` in `.gitignore`**. Als Vorlage eine `.dev.vars.example` ohne Werte committen.
- In CI muss eventuell `CLOUDFLARE_INCLUDE_PROCESS_ENV=true` gesetzt sein, damit Variablen beim Build verfügbar sind [2]. Für Deploys aus GitHub Actions braucht es ein `CLOUDFLARE_API_TOKEN` als GitHub-Actions-Secret (nie im Repo).
- `wrangler.jsonc` enthält D1-Binding-Namen und die `database_id`. Die ID allein gewährt keinen Zugriff. Wer sie trotzdem nicht veröffentlichen will, kann sie über Umgebungen bzw. eine lokale Config auslagern (Ermessensfrage, nicht in den Quellen geregelt).

**Vercel / Netlify:** Env-Variablen im Projekt-Dashboard bzw. per CLI setzen. Das TanStack-Start-Prinzip (kein `VITE_`-Prefix, Zugriff nur serverseitig) gilt genauso.

**Bon-Scan-Server-Function:** Auf Cloudflare läuft der Aufruf der Vision-API als `fetch()` aus der Server Function. Die Wartezeit zählt nicht als CPU [4]. Der Request-Body darf bis 100 MB groß sein. Auf Vercel liegt die Grenze bei 4,5 MB [7]. In beiden Fällen sollte das Foto im Client verkleinert werden (schneller und günstiger bei der Vision-API). Da Bon-Fotos nicht dauerhaft gespeichert werden sollen, wird kein R2/Blob-Speicher gebraucht.

## 6. Empfehlung

### A) Cloudflare Workers + D1 (empfohlen)
- **Kosten:** 0 € im Free Tier. Falls das CPU-Limit von 10 ms stört, Workers Paid für 5 $/Monat, dann sind auch die D1-Kontingente deutlich höher.
- **Warum:** offizieller TanStack-Start-Partner [1], kein Pausieren und kein Cold Start der DB, D1 als Binding ohne Verbindungs-Secret, Time Travel 7 Tage als Backup-Ersatz [16], Secrets nicht mehr auslesbar [28]. Static Assets sind kostenlos [5].
- **TanStack DB:** QueryCollection über Server Functions (+ Offline-Transaktionen/Persistenz im Client). Electric und PowerSync sind damit ausgeschlossen.
- **Risiken:** 10 ms CPU im Free Tier. Daten in SQLite statt Postgres (für Snack unproblematisch). Vendor-Lock-in bei D1-Bindings. Mit Drizzle o. ä. bleibt ein Umzug nach Turso/SQLite aber machbar.

### B) Cloudflare Workers + Supabase Free + Electric Cloud (nur wenn das TanStack-DB-Ticket Electric wählt)
- **Kosten:** 0 € (Electric unter 5 $ erlassen, Supabase Free).
- **Warum:** Postgres mit Logical Replication für die ElectricCollection. Die Supabase-Compute läuft ohnehin, also keine Zusatzkosten durch die Dauerverbindung.
- **Risiken:** Supabase pausiert nach 7 Tagen Inaktivität, danach manuell wiederherstellen [20]. Drei Anbieter und mehr Secrets (DB-URL, Electric-Credentials). Von Workers zu Postgres über Hyperdrive (Free: 100.000 Queries/Tag) [29] oder über die HTTP-API von Supabase. **Neon statt Supabase** nur ohne Dauer-Sync, siehe Rechnung in Abschnitt 3.

**Nicht empfohlen:** Netlify Free (Seite pausiert bei Credit-Ende, Deploys kosten Credits), Railway (5 $ Grundgebühr, Dauerbetrieb übersteigt das Guthaben), Fly.io (kein Free Tier, Postgres ab 38 $), PowerSync Cloud (Deaktivierung im Free Tier, Pro 49 $).

## Offene Punkte

- Ob die 10 ms CPU im Free Tier für SSR von Snack reichen, lässt sich nur mit einem Prototyp messen (`wrangler dev` / Dashboard-Metriken).
- Netlify-Function-Limits (Payload, Timeout) ließen sich auf den abgerufenen Seiten nicht belegen und wurden deshalb nicht aufgenommen.
- Die kleinste Neon-Compute im Free Tier (0,25 CU) ist eine Annahme.
- Turso: ob inaktive Free-Datenbanken archiviert werden, steht nicht auf der Pricing-Seite.
- Absicherung der App (Auth für einen einzigen Nutzer) ist nicht Teil dieses Tickets.

## Quellen (abgerufen 2026-10-30)

1. TanStack Start – Hosting: https://tanstack.com/start/latest/docs/framework/react/guide/hosting
2. Cloudflare – TanStack Start on Workers: https://developers.cloudflare.com/workers/framework-guides/web-apps/tanstack-start/
3. Cloudflare Workers – Pricing: https://developers.cloudflare.com/workers/platform/pricing/
4. Cloudflare Workers – Limits: https://developers.cloudflare.com/workers/platform/limits/
5. Cloudflare Workers – Static Assets Billing: https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/
6. Vercel – Pricing: https://vercel.com/pricing
7. Vercel – Functions Limits: https://vercel.com/docs/functions/limitations
8. Netlify – Pricing: https://www.netlify.com/pricing/
9. Netlify – Credit-based pricing plans: https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/credit-based-pricing-plans/
10. Netlify – Resume paused projects: https://docs.netlify.com/manage/accounts-and-billing/billing/resume-paused-projects/
11. Railway – Pricing: https://railway.com/pricing
12. Railway – App Sleeping: https://docs.railway.com/reference/app-sleeping
13. Railway – Free Trial: https://docs.railway.com/reference/pricing/free-trial
14. Fly.io – Pricing: https://docs.fly.io/about/pricing
15. Cloudflare D1 – Pricing: https://developers.cloudflare.com/d1/platform/pricing/
16. Cloudflare D1 – Limits: https://developers.cloudflare.com/d1/platform/limits/
17. Neon – Pricing: https://neon.com/pricing
18. Neon – Logical Replication: https://neon.com/docs/guides/logical-replication-neon
19. Supabase – Pricing: https://supabase.com/pricing
20. Supabase – Project Pausing: https://supabase.com/docs/guides/platform/free-project-pausing
21. Turso – Pricing: https://turso.tech/pricing
22. TanStack DB – Overview: https://tanstack.com/db/latest/docs/overview
23. TanStack DB – Query Collection: https://tanstack.com/db/latest/docs/collections/query-collection
24. TanStack DB – Electric Collection: https://tanstack.com/db/latest/docs/collections/electric-collection
25. PowerSync – Pricing: https://www.powersync.com/pricing
26. Electric – Pricing: https://electric.ax/pricing
27. TanStack Start – Environment Variables: https://tanstack.com/start/latest/docs/framework/react/guide/environment-variables
28. Cloudflare Workers – Secrets: https://developers.cloudflare.com/workers/configuration/secrets/
29. Cloudflare Hyperdrive – Pricing: https://developers.cloudflare.com/hyperdrive/platform/pricing/
