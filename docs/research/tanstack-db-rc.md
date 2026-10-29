# Research: TanStack DB RC für Offline-Erfassung in Snack

Stand: 2026-10-30. Beantwortet Issue #2 („TanStack DB RC: tragfähig für Offline-Erfassung, und welcher Sync-/Persistenz-Weg passt?“).
Quellen sind ausschließlich Primärquellen (offizielle Doku im Repo `TanStack/db`, TanStack-Blog, npm-Registry, GitHub-Issues, Hersteller-Doku). Jede Aussage ist mit einer Quelle aus der Liste am Ende verknüpft: [Sx].

## Kurzantwort

**Ja, mit einem klar begrenzten Ausschnitt.** Für Snack reicht dieser minimale Weg:
`queryCollectionOptions` (Server bleibt Source of Truth, eigene API in TanStack Start) + `@tanstack/offline-transactions` (dauerhafte Outbox in IndexedDB, Retry, Wiederherstellung des optimistischen Zustands nach Reload), App-Routen **ohne SSR** (`ssr: false` / SPA-Modus).
**Nicht** nötig sind ein Sync-Engine (ElectricSQL, PowerSync) oder die Browser-SQLite/OPFS-Persistenz in v1. Die SQLite-Persistenz lässt sich später nachrüsten, falls die Übersicht auch offline nach einem Kaltstart sichtbar sein soll.
Vorher kommt ein kurzer Prototyp (Spike), der das Offline-Reload-Verhalten auf dem echten Handy bestätigt (siehe „Offene Punkte“). Als Fallback bleibt reines TanStack Query mit persistierten, pausierten Mutations. Der Server-Vertrag (idempotentes Upsert per Client-UUID) ist in beiden Fällen gleich, ein Wechsel ist also billig.

## 1. Reifegrad der RC

- `@tanstack/db@1.0.0-rc.1` ist seit **2026-10-29** veröffentlicht und steht auf npm sowohl unter `rc` als auch unter `latest`. Vorher kamen 0.15.0–0.15.2 im Oktober 2026 [S1]. Die RC ist bei Recherche also **einen Tag alt**.
- Die Ankündigung „TanStack DB 1.0 RC1: Going All In on Oracles“ (2026-10-29) nennt die API „powerful, production-ready, and stable“. Seit 0.6 neu: SSR mit „deep integration with TanStack Start“, Adapter für Neon Realtime und Convex, Preact, viele Fixes. Grundlage sind rund 3 Monate Härtung mit Oracle-/Property-based Tests: 211 Oracle-Testdateien über 15 Pakete, TLA+-Modelle für schwierige Designs [S2].
- Die Ankündigung räumt ein, dass weiterhin Regressionen und Edge-Case-Bugs gefunden werden. Einen Termin für 1.0 stable nennt sie nicht [S2].
- Die README auf `main` zeigt noch das Badge „status-beta“ und den Satz „Tanstack DB is currently in BETA“ [S3]. Die Doku ist in diesem Punkt nicht nachgezogen.
- **Versions-Eigenheit:** `@tanstack/query-db-collection` (1.5.0) und `@tanstack/offline-transactions` (1.2.0) wurden laut Migrationsguide „accidentally released as 1.0 before the RC“. Sie bleiben auf 1.x und verlangen `@tanstack/db@^1.0.0-rc.0` als Peer [S4]. Nur die Major-Nummer dieser beiden Pakete wirkt also stabil. Ihr Reifegrad entspricht dem der RC.
- Der Migrationsguide von 0.9.2 auf die RC listet viele Breaking Changes: `$synced` heißt jetzt `$hasPendingWrites`, `isPersisted.promise` heißt jetzt `when('settled')`, Query-Collection-Handler refetchen nicht mehr automatisch, das Dependency-Array in `useLiveQuery` ist entfernt [S4]. Bis kurz vor der RC war die API also noch stark in Bewegung.
- Aktivität: 136 offene Issues/PRs, letzter Push 2026-10-29 [S5]. Die Core-Pakete haben laut Blog über eine Million wöchentliche npm-Downloads. Als Nutzer genannt werden Prisma Studio, Unkey und Skylight [S2].

**Einschätzung:** Kern-API (Collections, Live Queries, optimistische Mutations) und Query Collection sind reif genug für eine private App. Persistenz- und Multi-Tab-Pfade sind deutlich jünger (siehe 5.).

## 2. Collection-Typen und Sync-Optionen

Laut `docs/collections` [S6]:

| Typ | Paket | Sync | Offline-Relevanz für Snack |
| --- | --- | --- | --- |
| Query Collection | `@tanstack/query-db-collection` | Lädt per TanStack Query `queryFn`, schreibt per `onInsert`/`onUpdate`/`onDelete` an die eigene API [S7] | **Empfohlen.** Passt zu Query/Start, kein Zusatzdienst |
| IndexedDB Collection | `@tanstack/db` (Core, **neu seit 0.9.2**) [S4] | Rein lokal, „does not synchronize data to a server“ [S8] | Lokaler Speicher ohne Server-Sync. Sync müsste man selbst schreiben |
| LocalStorage Collection | `@tanstack/db` | Lokal, Tab-Sync über storage events [S9] | Für kleine Präferenzen gedacht |
| LocalOnly Collection | `@tanstack/db` | Nur In-Memory [S8] | Nur UI-Zustand |
| Electric Collection | `@tanstack/electric-db-collection` | Postgres → Shapes, Echtzeit | Braucht Postgres + Electric-Dienst |
| PowerSync Collection | `@tanstack/powersync-db-collection` | Sync-Service + lokale SQLite | Braucht PowerSync-Dienst |
| RxDB, TrailBase, Convex, Neon Realtime | jeweilige Pakete | Fremd-Backends | Nicht relevant |

Querschnitt dazu:

- **SQLite-Persistenz** (`persistedCollectionOptions`) umschließt jede Collection, mit oder ohne `sync`, und speichert ihre Zeilen in SQLite. Im Browser ist das `@tanstack/browser-db-sqlite-persistence` mit wa-sqlite in einem Worker auf **OPFS** [S10].
- **Offline Transactions** (`@tanstack/offline-transactions`) sind eine dauerhafte Outbox für noch nicht bestätigte Mutations [S11].

Die Doku trennt die beiden ausdrücklich: „The outbox stores pending writes. SQLite persistence stores Collection rows.“ [S11]

### Sync-Engines und Kosten (Kostenrahmen < ~5 €/Monat)

- **ElectricSQL Cloud:** PAYG ohne Grundgebühr, „Under $5/mo waived“, $1 pro 1M Writes, Postgres-Sync +$2 pro 1M Writes [S12]. Für einen Nutzer wohl kostenlos. Zusätzlich braucht es aber Postgres, einen Proxy für Shapes und `txid`-Handling. Die Electric+Start-Integration hat laut Skill-Feedback-Issue Doku-Lücken auf der Serverseite [S13].
- **PowerSync Cloud:** Free Tier mit 2 GB Sync pro Monat und 500 MB Storage. Wird nach 1 Woche Inaktivität deaktiviert (die Seite widerspricht sich, an anderer Stelle stehen 2 Wochen). Pro ab $49/Monat. Self-Hosting als „Open Edition“ möglich [S14]. Die Inaktivitäts-Abschaltung passt schlecht zu einer App, die man mal eine Woche nicht öffnet. Bekannter Bug: `$synced` spiegelte nur die lokale Persistenz, nicht den Upload-Status [S15].
- **Fazit:** Beide lösen Mehrgeräte-Echtzeit-Sync und Konfliktfälle. Diese Probleme hat ein Einzelnutzer, der fast nur anhängt („Snack erfasst“), kaum. Sie kosten zusätzliche Infrastruktur und Komplexität ohne nennenswerten Nutzen.

## 3. Offline-Verhalten optimistischer Mutations und Persistenz über Reload

### Ohne Zusatzpaket (nur Collection + Handler)

- Optimistische Änderungen wirken sofort. Wirft der Handler (z. B. `fetch` offline), werden sie **automatisch zurückgerollt** („Failed transactions automatically rollback their optimistic state“) [S16].
- Optimistischer Zustand liegt nur im Speicher. Für Mutations, die einen App-Neustart überstehen sollen, verweist die Doku auf Offline Transactions [S16].
- **Offline erfasste Snacks gingen ohne Outbox also verloren oder würden zurückgerollt.** Für Snack ist das nicht akzeptabel.

### Mit `@tanstack/offline-transactions` [S11][S17]

- `startOfflineExecutor({ collections, mutationFns })` plus `createOfflineAction({ mutationFnName, onMutate })`. `onMutate` muss synchron sein und wendet die optimistische Änderung an. Danach schreibt der Executor die Transaktion in die Outbox.
- Speicher im Web: zuerst **IndexedDB**, Fallback localStorage. Ein eigener `storage`-Adapter ist möglich [S11].
- „The leader restores their optimistic state after a restart.“ Nach einem Reload erscheinen ausstehende Snacks also wieder in der UI und werden später gesendet [S11].
- Retry mit exponentiellem Backoff und Jitter. Die Abarbeitung ist **FIFO und seriell**. `NonRetriableError` markiert endgültige Fehler, die zurückgerollt werden. Mit `shouldRetry` lässt sich das Retry-Verhalten anpassen [S11][S17].
- **Idempotenz ist Pflicht:** Der Executor schickt einen stabilen `idempotencyKey`, und ein Request kann mehrfach ankommen, besonders nach Neustart oder Leader-Wechsel [S11]. Für Snack heißt das: Die Client-ID (`crypto.randomUUID()`) ist der Primärschlüssel, der Server macht ein Upsert per ID.
- Feinheiten [S11]:
  - Ein sichtbarer optimistischer Eintrag beweist noch nicht, dass die Outbox-Schreibung gelungen ist.
  - `transaction.when('settled')` kann offline beliebig lange offen bleiben. Man darf darauf nicht vor dem Rendern warten.
  - `executor.isOfflineEnabled`, `mode` und `storageDiagnostic` nach `waitForInit()` prüfen. Ohne Leadership oder ohne Storage fällt der Executor stumm auf „online-only“ zurück.
- Mehrere Tabs: Nur der Leader (gewählt über Web Locks, sonst BroadcastChannel) hat die Outbox. Follower-Tabs mutieren nur online [S11]. Eine installierte PWA läuft praktisch in einem Fenster, das ist hier unkritisch.
- Erkennung von Online/Offline: `online`- und `visibilitychange`-Events. Sichtbare Tabs versuchen es auch dann, wenn `navigator.onLine` false meldet [S17].

### Zeilen-Persistenz (Lesen offline nach Kaltstart)

- Die Outbox stellt nur **ausstehende** Snacks wieder her. Bereits synchronisierte Snacks (Historie, Übersicht) liegen nach einem Kaltstart offline **nicht** vor, solange die Query Collection nicht laden kann.
- Lösung laut Doku: Query Collection in `persistedCollectionOptions` mit `@tanstack/browser-db-sqlite-persistence` einwickeln (OPFS, wa-sqlite-WASM). Dazu `initialRender: { strategy: 'network-first', networkTimeoutMs }`, damit die Collection aus den wiederhergestellten Zeilen `ready` wird, wenn das Netz nicht antwortet [S10].
- Reifegrad: Im 0.6-Post (2026-03) hieß das noch „the first _alpha_ release of persistence“. Damals wurde bewusst gegen ein eigenes IndexedDB-Backend und für SQLite überall entschieden [S18]. In der RC kam `initialRender` neu dazu [S4]. Die RC liefert jetzt die `wa-sqlite.wasm` getrennt vom Worker aus. In Vite kann `optimizeDeps.exclude` nötig sein [S4].
- Alternative ohne WASM: die IndexedDB Collection im Core. Sie lädt den ganzen Store in den Speicher und hat **keinen** Server-Sync [S8]. Snack müsste dann einen eigenen Push/Pull-Mechanismus schreiben. Das ist mehr Eigenbau als der empfohlene Weg.
- Browser-Speicher ist „best-effort“ und kann verdrängt werden. WebKit gewährt `navigator.storage.persist()` nach Heuristiken, etwa wenn die Seite als Home-Screen-Web-App geöffnet ist [S19]. Für Snack: als PWA installieren und `persist()` anfragen. Die Server-Kopie bleibt das Backup.
- **Offline-Start der App** (HTML/JS aus dem Cache) ist nicht Teil von TanStack DB. „Caching the app for offline startup … require[s] separate setup“ [S8], also einen Service Worker. Das gehört zum Map-Punkt „PWA-Details“.

## 4. Integration mit TanStack Start (SSR)

- Seit der RC gibt es echtes SSR: `DbClient` pro Request, `collectionOptions(id, factory)`-Deskriptoren, `DbProvider`, `dehydrate()`/`hydrate()` und `@tanstack/react-router-with-db` mit `routerWithDbClient(router, dbClient)` für Suspense-Streaming [S20].
- SSR ist **optional**. `createCollection(...)` und Collection-Instanzen in `useLiveQuery` bleiben unterstützt [S20].
- SSR-Fallstricke [S4][S20]:
  - Rendern mit Zeilen erfordert ein passendes `preloadLiveQuery()`. Sonst wirft die RC einen Fehler mit Hinweis.
  - Indizes müssen vor der Query-Kompilierung angelegt werden.
  - Modul-Singletons sind auf dem Server nicht request-isoliert.
  - Offener Bug: `DbClient.cleanup()` loggt bei verketteten Live Queries auf **jedem** SSR-Request einen Fehler [S21].
- **Offline und SSR passen nicht zusammen.** IndexedDB, OPFS und die Outbox existieren nur im Browser, und offline gibt es keinen Server, der rendert. Das offizielle Beispiel `examples/react/offline-transactions` ist eine TanStack-Start-App (`@tanstack/react-start`, `query-db-collection`, `offline-transactions`, `browser-db-sqlite-persistence`). Es läuft mit `tanstackStart({ spa: { enabled: true } })` und startet den Executor im Browser in einem `useEffect` [S22].
- TanStack Start erlaubt das auch pro Route: `ssr: false` überspringt `beforeLoad`, `loader` und das Rendern auf dem Server. `defaultSsr: false` in `createStart` schaltet SSR für die ganze App ab [S23].
- Bewährtes Muster laut Nutzer-Feedback: `ssr: false` + `await collection.preload()` im Route-Loader [S13].
- **Für Snack:** App-Routen client-only (SPA-Modus oder `defaultSsr: false`). Server-Routen bzw. `createServerFn` von Start dienen als API (`GET /snacks`, idempotentes `PUT /snacks/:id`). Die SSR-APIs von TanStack DB werden nicht gebraucht.

## 5. Bekannte Grenzen und Bugs (Auswahl offener Issues, Stand 2026-10-30)

| Bereich | Issue | Relevanz für Snack |
| --- | --- | --- |
| Offline-Transactions: Optimistischer Zustand fällt sofort nach `mutationFn` weg. Bis der Sync nachzieht, kann die Zeile flackern. Abwarten im `mutationFn` blockiert die serielle Abarbeitung | #1602 [S24] | Gering: Die Query Collection wird im `mutationFn` per `refetch()` oder Direct Write aktualisiert, ohne Electric-`awaitTxId` |
| `OfflineExecutor.dispose()` lässt `pendingTransactionPromises` hängen | #2254 [S25] | Gering: nur beim Neustart des Executors (z. B. HMR, Unmount) |
| `network-first`: Eine eager Collection, die zuletzt 0 Zeilen synchronisiert hat, bleibt offline `loading` (dokumentiertes Verhalten) | #2284 [S26] | Mittel: nur bei Zeilen-Persistenz und leerer Historie, z. B. am ersten Tag |
| SQLite-Persistenz: Fehlgeschlagene dauerhafte Schreibung ändert trotzdem die öffentliche Collection | #2210 [S27] | Nur bei optionaler Zeilen-Persistenz |
| Browser-SQLite Multi-Tab: Schließen des Leader-Tabs bringt Follower mit `IndeterminateCommitError` zum Stillstand. Weitere Follower-Bugs | #2085, #2139, #2174 [S27] | Gering: PWA mit einem Fenster |
| On-demand-Caches sind „not yet a certified offline snapshot“ | Doku [S10], #2102 [S24] | Nicht relevant bei eager Sync (Snack-Datenmenge klein) |
| SSR-Cleanup-Fehler bei verketteten Live Queries | #2293 [S21] | Nicht relevant bei `ssr: false` |

Dazu kommen dokumentierte Grenzen:

- Die IndexedDB Collection lädt den gesamten Store in den Speicher [S8]. Bei Snack-Mengen (einige Tausend Zeilen pro Jahr) ist das unkritisch.
- Bei `schemaVersion`-Wechsel ohne `sync` wirft die Collection, statt Daten zu löschen [S10]. Mit `sync` werden die lokalen Daten zurückgesetzt und neu geladen.
- `offline-transactions` brauchen stabile `mutationFnName`s und Collection-IDs über Releases hinweg, sonst bleiben Outbox-Einträge liegen [S11].

## 6. Minimaler Weg für Snack (Empfehlung)

1. **Pakete:** `@tanstack/react-db@rc`, `@tanstack/query-db-collection@^1.5`, `@tanstack/offline-transactions@^1.2`, Versionen exakt pinnen. Kein Electric, kein PowerSync, in v1 keine Browser-SQLite.
2. **Server (TanStack Start):** `GET /api/snacks` und `PUT /api/snacks/:id` als idempotentes Upsert per Client-UUID. `Idempotency-Key` kann zusätzlich mitgeschickt werden, die UUID reicht aber als Schlüssel. Die Datenbank wählt das Hosting-Ticket.
3. **Client:** eine `snacks`-Collection mit `queryCollectionOptions` (stabile `id`, `getKey: (s) => s.id`, Zod-Schema). Dazu ein `startOfflineExecutor({ collections: { snacks }, mutationFns: { saveSnack } })`, beim App-Start im Browser initialisiert.
   - Erfassen läuft über `createOfflineAction({ mutationFnName: 'saveSnack', onMutate: (snack) => snacks.insert(snack) })`.
   - `saveSnack` macht den `PUT` und danach `await snacks.utils.refetch()` (oder einen Direct Write).
   - Bei 4xx-Validierungsfehlern wirft `saveSnack` einen `NonRetriableError`.
4. **SSR aus** für App-Routen (`defaultSsr: false` oder SPA-Modus), so wie im offiziellen Beispiel [S22][S23].
5. **UI:** `$hasPendingWrites` an jedem Snack und `executor.getPendingCount()` zeigen „noch nicht synchronisiert“ an [S4][S11]. Nach `waitForInit()` `isOfflineEnabled` prüfen und bei `false` einen Hinweis zeigen.
6. **Später/optional:**
   - Für eine offline sichtbare Übersicht nach Kaltstart die Collection in `persistedCollectionOptions` + `createBrowserWASQLitePersistence` mit `initialRender: { strategy: 'network-first', networkTimeoutMs: 0 }` einwickeln [S10].
   - Für den Offline-App-Start einen Service Worker einrichten (PWA-Ticket).
7. **Fallback, falls die RC im Spike enttäuscht:** reines TanStack Query mit `persistQueryClient` und pausierten Mutations. Dafür eine Default-`mutationFn` per `setMutationDefaults` registrieren und `resumePausedMutations()` in `onSuccess` von `PersistQueryClientProvider` aufrufen [S28]. Server-Vertrag und Client-UUIDs bleiben identisch.

### Skizze (ohne Typecasts)

```ts
import { createCollection } from '@tanstack/react-db'
import { queryCollectionOptions } from '@tanstack/query-db-collection'
import { NonRetriableError, startOfflineExecutor } from '@tanstack/offline-transactions'
import { z } from 'zod'

const snackSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  priceCents: z.number().int().nonnegative(),
  store: z.string().optional(),
  category: z.string().optional(),
  quantity: z.number().int().positive(),
  date: z.string(), // ISO-Datum
})

export const snacks = createCollection(
  queryCollectionOptions({
    id: 'snacks',
    queryClient,
    queryKey: ['snacks'],
    queryFn: async () => z.array(snackSchema).parse(await (await fetch('/api/snacks')).json()),
    getKey: (snack) => snack.id,
    schema: snackSchema,
  }),
)

export const executor = startOfflineExecutor({
  collections: { snacks },
  mutationFns: {
    saveSnack: async ({ transaction, idempotencyKey }) => {
      const snack = snackSchema.safeParse(transaction.mutations[0]?.modified)
      if (!snack.success) throw new NonRetriableError('Ungültiger Snack')
      const res = await fetch(`/api/snacks/${snack.data.id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json', 'Idempotency-Key': idempotencyKey },
        body: JSON.stringify(snack.data),
      })
      if (res.status === 400 || res.status === 422) throw new NonRetriableError(`HTTP ${res.status}`)
      if (!res.ok) throw new Error(`HTTP ${res.status}`) // wird erneut versucht
      await snacks.utils.refetch()
    },
  },
})
```

(Skizze, nicht kompiliert. Sie lehnt sich an die Beispiele in [S11] und [S22] an. `queryClient` kommt aus dem bestehenden TanStack-Query-Setup.)

## Offene Punkte für den Prototyp-Spike

Die Doku beantwortet folgende Frage nicht eindeutig: Was zeigt die Query Collection nach einem **Offline-Kaltstart** ohne Zeilen-Persistenz an? Ein fehlgeschlagener oder pausierter `queryFn` lässt die Collection vermutlich in `loading`/`error`. Die Outbox stellt die ausstehenden Snacks optimistisch wieder her.

Der Spike sollte auf iOS/Android als installierte PWA prüfen:

1. Offline erfassen
2. Reload/Kaltstart offline
3. Ausstehende Snacks sichtbar? Erfassung möglich?
4. Wieder online: genau ein Server-Datensatz pro Snack?

Fällt Punkt 3 schlecht aus, gibt es zwei Wege: die optionale SQLite-Persistenz sofort einbauen oder auf den TanStack-Query-Fallback wechseln.

## Quellen

- [S1] npm-Registry `@tanstack/db` (dist-tags, Zeitstempel): https://www.npmjs.com/package/@tanstack/db?activeTab=versions
- [S2] TanStack Blog, „TanStack DB 1.0 RC1: Going All In on Oracles“ (2026-10-29): https://tanstack.com/blog/tanstack-db-1.0-rc1-going-all-in-on-oracles
- [S3] TanStack/db README: https://github.com/TanStack/db/blob/main/README.md
- [S4] Migrating to the 1.0 RC: https://github.com/TanStack/db/blob/main/docs/guides/migrate-to-1-0-rc.md
- [S5] GitHub-Repo TanStack/db: https://github.com/TanStack/db
- [S6] Collections-Doku (Verzeichnis): https://github.com/TanStack/db/tree/main/docs/collections
- [S7] Query Collection: https://github.com/TanStack/db/blob/main/docs/collections/query-collection.md
- [S8] IndexedDB Collection: https://github.com/TanStack/db/blob/main/docs/collections/indexed-db-collection.md
- [S9] LocalStorage Collection: https://github.com/TanStack/db/blob/main/docs/collections/local-storage-collection.md
- [S10] SQLite Persistence Guide: https://github.com/TanStack/db/blob/main/docs/guides/sqlite-persistence.md
- [S11] Offline Transactions Guide: https://github.com/TanStack/db/blob/main/docs/guides/offline-transactions.md
- [S12] Electric Cloud Pricing: https://electric.ax/pricing
- [S13] Issue #1336 „Skill Feedback: meta-framework“: https://github.com/TanStack/db/issues/1336
- [S14] PowerSync Pricing: https://www.powersync.com/pricing
- [S15] Issue #1526 „[PowerSync]: $synced reflects local SQLite persistence“: https://github.com/TanStack/db/issues/1526
- [S16] Mutations Guide: https://github.com/TanStack/db/blob/main/docs/guides/mutations.md
- [S17] `@tanstack/offline-transactions` README: https://github.com/TanStack/db/blob/main/packages/offline-transactions/README.md
- [S18] TanStack Blog, „TanStack DB 0.6 Now Includes Persistence, Offline Support, and Hierarchical Data“ (2026-03-25): https://tanstack.com/blog/tanstack-db-0.6-app-ready-with-persistence-and-includes
- [S19] WebKit Blog, „Updates to Storage Policy“: https://webkit.org/blog/14403/updates-to-storage-policy/
- [S20] SSR and Hydration Guide: https://github.com/TanStack/db/blob/main/docs/guides/ssr.md
- [S21] Issue #2293 „DbClient.cleanup() … logging an error on every SSR request“: https://github.com/TanStack/db/issues/2293
- [S22] Offizielles Beispiel (TanStack Start, SPA-Modus): https://github.com/TanStack/db/tree/main/examples/react/offline-transactions
- [S23] TanStack Start, Selective SSR: https://tanstack.com/start/latest/docs/framework/react/guide/selective-ssr
- [S24] Issues #1602 und #2102: https://github.com/TanStack/db/issues/1602, https://github.com/TanStack/db/issues/2102
- [S25] Issue #2254: https://github.com/TanStack/db/issues/2254
- [S26] Issue #2284: https://github.com/TanStack/db/issues/2284
- [S27] Issues #2210, #2085, #2139, #2174: https://github.com/TanStack/db/issues/2210, https://github.com/TanStack/db/issues/2085, https://github.com/TanStack/db/issues/2139, https://github.com/TanStack/db/issues/2174
- [S28] TanStack Query, Mutations: Persisting Offline mutations: https://tanstack.com/query/latest/docs/framework/react/guides/mutations
