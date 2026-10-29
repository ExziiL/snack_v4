# Research: Absicherung für einen einzelnen Nutzer

> Ticket: [#4 – Absicherung für einen einzelnen Nutzer: Wie komme nur ich in die App?](https://github.com/ExziiL/snack_v5/issues/4) (Teil der Map [#1](https://github.com/ExziiL/snack_v5/issues/1))
> Stand: 2026-10-30. Quellen sind Primärquellen (offizielle Docs, Specs, Hersteller-Blogs), außer wo ausdrücklich als **Community-Bericht** markiert.

## Frage

Welche Optionen gibt es, eine TanStack-Start-App so abzusichern, dass nur genau eine Person Zugriff hat? Vergleich nach Sicherheit, Bequemlichkeit als installierte PWA auf dem Handy (lange Sessions, kein ständiges Einloggen), Offline-Verhalten, Kosten und Aufwand.

## Kurzantwort

**Empfehlung: Auth in der App mit [Better Auth](https://www.better-auth.com/) + Passkey-Plugin**, Einzelnutzer erzwungen per `databaseHooks.user.create.before` (nur `OWNER_EMAIL` aus einer Umgebungsvariable, nur solange noch kein User existiert) und danach `disableSignUp`. Die Session ist ein first-party, server-gesetztes HttpOnly-Cookie mit langer, gleitender Laufzeit (z. B. `expiresIn` 180 Tage, `updateAge` 1 Tag). Login im Alltag: Face ID / Fingerabdruck per Passkey, ein Tipp. Ein starkes Passwort aus dem Passwortmanager dient als Fallback.

Vorgeschaltete Plattform-Gates (Cloudflare Access, Vercel Authentication, Netlify private Projekte) kosten 0 € und null Code. Für eine *installierte Offline-PWA* sind sie aber schlechter: höchstens 1 Monat Session (Access), Redirects auf fremde Domains (problematisch im iOS-Standalone-Modus), Service-Worker- und Manifest-Requests, die an der Auth scheitern, und Bindung an einen Hoster. Sie eignen sich gut als **Zusatzschutz für Preview-Deployments**, nicht als Haupt-Login.

## Rahmenbedingungen, die die Bewertung treiben

### Cookies und Speicher als installierte PWA

- **iOS/Safari, ITP:** Cookies, die per JavaScript (`document.cookie`) gesetzt werden, und *script-writable storage* (IndexedDB, LocalStorage, Service-Worker-Registrierung und -Cache) löscht ITP nach 7 Tagen ohne Nutzerinteraktion. Server-gesetzte first-party Cookies fallen **nicht** unter diese Löschung. Die Ausnahme sind Cookies aus Antworten von *CNAME- oder IP-getarnten Drittanbietern*: Deren Ablauf wird auf 7 Tage gekappt. – [WebKit Tracking Prevention](https://webkit.org/tracking-prevention/), [MDN Storage quotas and eviction](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria) („Cookies set by server are exempt from this eviction.“)
- **Home-Screen-Web-Apps sind ausgenommen:** „The first-party domain of home screen web applications is exempt from ITP's 7-day cap.“ Sie haben eigene Website-Daten, getrennt von Safari. – [WebKit Tracking Prevention](https://webkit.org/tracking-prevention/), [WebKit-Blog „Full Third-Party Cookie Blocking and More“](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/)
  - **Folge:** Auf iOS hat die installierte PWA einen **eigenen Cookie-Speicher**, getrennt von Safari. Ein Login in Safari zählt in der Home-Screen-App nicht. Der Login muss *in* der installierten App stattfinden.
- **Persistenter Speicher:** `navigator.storage.persist()` nimmt eine Origin von der Eviction aus. WebKit gewährt das heuristisch, u. a. danach, „whether the website is opened as a Home Screen Web App“. – [WebKit „Updates to Storage Policy“](https://webkit.org/blog/14403/updates-to-storage-policy/). Das ist relevant für die Offline-Daten (TanStack DB/IndexedDB), nicht für das Session-Cookie.
- **Safari-Bug (behoben in 27.2):** Bis Safari 27.2 wurden Cookies aus der Antwort einer *Top-Level-Navigation* fälschlich auf 7 Tage gekappt, wenn der Hostname per CNAME auf eine andere registrierbare Domain zeigt (typisch bei Custom Domains auf PaaS). – [WebKit Features for Safari 27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/). Redirect-basierte Logins (OAuth-Callback, Access-Redirect) setzen das Cookie genau so. Ein Passkey-Login setzt das Cookie per `fetch`-Antwort und ist davon nicht betroffen.
- **Android/Chrome:** Eine installierte PWA (WebAPK) nutzt das Chrome-Profil, Cookies und Storage werden mit dem Browser geteilt. – [web.dev: WebAPKs on Android](https://web.dev/webapks). Chrome kappt Cookie-Lebensdauer auf **400 Tage** (seit M104). – [Chrome for Developers: Cookie Expires and Max-Age attributes now have upper limit](https://developer.chrome.com/blog/cookie-max-age-expires)

### Passkeys / WebAuthn

- iOS: synced Passkeys ab iOS 16, Conditional UI (Autofill) ab iOS 16.1. Android: synced Passkeys ab Android 9, Conditional UI in Chrome 108+. – [passkeys.dev Device Support](https://passkeys.dev/device-support/)
- WebKit verlangt für Face ID / Touch ID bei WebAuthn eine **User-Geste**: `navigator.credentials.*` muss aus einem Tap-Handler kommen. Eine über `fetch` weitergereichte Geste läuft nach 10 s ab. – [WebKit „Meet Face ID and Touch ID for the Web“](https://webkit.org/blog/11312/meet-face-id-and-touch-id-for-the-web/). **Folge:** Kein automatischer Login beim App-Start. Es braucht einen Button „Mit Passkey anmelden“, und vor `signIn.passkey()` darf kein langer Request laufen.
- **Offene Lücke:** Keine Apple-Primärquelle sagt ausdrücklich, dass WebAuthn im iOS-Standalone-Modus (Home-Screen-Web-App) geht. Die Safari-Engine ist dieselbe und Passkeys liegen im System-Schlüsselbund. In der Praxis wird es breit genutzt, es ist hier aber **nicht belegt**. → Vor der Festlegung einmal auf dem eigenen iPhone testen (s. Prototyp-Hinweis unten).
- Passkeys werden über iCloud-Schlüsselbund bzw. Google Password Manager synchronisiert. Ein in Safari registrierter Passkey ist damit auch in der Home-Screen-App für dieselbe RP-ID nutzbar. Das ist eine Ableitung aus dem „synced passkeys“-Modell, nicht explizit dokumentiert.

### Redirect-Logins im iOS-Standalone-Modus (Community-Berichte)

Mehrere Entwicklerberichte beschreiben OAuth- und Magic-Link-Probleme in iOS-Home-Screen-Apps: Magic Links öffnen sich in Safari statt in der App, und bei OAuth-Callbacks passt der `state` nicht, weil der Callback im falschen Cookie-Kontext landet. – **Community**: [Auth0 Community: Universal login broken on homescreen app](https://community.auth0.com/t/universal-login-broken-on-homescreen-app-in-ios-12-2/25787), [Auth0 Community: iOS add to home screen session](https://community.auth0.com/t/ios-add-to-home-screen-session-results-in-oops-something-went-wrong/11839). Die Berichte sind älter (iOS 11–12). Die Lage hat sich gebessert, gilt aber weiter als Fehlerquelle. **Folge:** Login-Verfahren ohne Wechsel auf eine fremde Domain (Passkey, Passwort, Code-Eingabe) sind in der PWA robuster als OAuth-Redirects und Magic Links.

## Optionen im Vergleich

### A) Better Auth in der App (Passkey + Passwort-Fallback) – **empfohlen**

- **TanStack-Start-Integration (offiziell):** Catch-all-Route `src/routes/api/auth/$.ts` ruft `auth.handler(request)` auf. Das Plugin `tanstackStartCookies()` (als **letztes** Plugin) lässt Cookies über Starts Cookie-System laufen. Session-Abfrage per `auth.api.getSession({ headers: getRequestHeaders() })`. Routen werden in `beforeLoad` (pathless Layout `_protected`) geschützt, Server-Funktionen per `ensureSession()`. – [Better Auth: TanStack Start Integration](https://www.better-auth.com/docs/integrations/tanstack)
- **Passkeys:** Paket `@better-auth/passkey`, Server-Plugin `passkey()`, Client-Plugin `passkeyClient()`, eigene Tabelle `passkey`. Login per `authClient.signIn.passkey()`, Autofill per `autocomplete="username webauthn"` + `signIn.passkey({ autoFill: true })`. `registration.requireSession` ist standardmäßig `true`: Einen Passkey registriert man erst nach einem anderen Login. – [Better Auth: Passkey Plugin](https://www.better-auth.com/docs/plugins/passkey)
- **Nur ein Nutzer:**
  - `emailAndPassword.disableSignUp` („Disable email and password sign up (default: false)“). Für Social-Provider gibt es `disableSignUp` / `disableImplicitSignUp`, dazu `disabledPaths` (z. B. `["/sign-up/email"]`). – [Better Auth: Options](https://www.better-auth.com/docs/reference/options)
  - `databaseHooks.user.create.before` kann die Anlage per `throw new APIError(...)` oder `return false` abbrechen. – [Better Auth: Database Hooks](https://www.better-auth.com/docs/concepts/database). Damit gilt: Einen User gibt es nur, wenn `email === process.env.OWNER_EMAIL` **und** noch kein User existiert. Das ist eine harte Einzelnutzer-Garantie, unabhängig vom Sign-up-Schalter.
  - Bootstrap: Direkt nach dem ersten Deploy einmal registrieren, dann `disableSignUp: true`. Solange Sign-up offen ist, könnte jemand, der die Owner-Mail kennt, zuvorkommen. Der Hook begrenzt das auf genau diesen einen Account. Alternativ ein Setup-Token aus einer Env-Variable verlangen.
- **Session-Laufzeit:** Standard `expiresIn` 7 Tage. `updateAge` verlängert die Session bei Nutzung (gleitend), `freshAge` ist standardmäßig 1 Tag (nur für sensible Aktionen). Ein optionaler `cookieCache` (signiert, `compact`/`jwt`/`jwe`) spart DB-Lookups. – [Better Auth: Session Management](https://www.better-auth.com/docs/concepts/session-management). Für Snack z. B. `expiresIn: 60*60*24*180`, `updateAge: 60*60*24`. Unter dem 400-Tage-Limit von Chrome bleibt die Session bei mindestens einem Gebrauch pro Halbjahr praktisch dauerhaft bestehen.
- **Datenbank:** Für Passkeys und widerrufbare Sessions nötig. Kysely-Adapter für SQLite/D1, Postgres, MySQL; Drizzle- und Prisma-Adapter. – [Better Auth: Database](https://www.better-auth.com/docs/concepts/database). Snack braucht ohnehin eine DB für den Sync, die Auth-Tabellen kommen mit dazu. (Ohne DB gibt es einen *stateless mode*, der passt aber nicht zu Passkeys.)
- **Sicherheit:** Hoch. Passkeys sind phishing-resistent, das Cookie ist HttpOnly und first-party. Sessions lassen sich serverseitig widerrufen. Die Auth-Logik liegt selbst in der eigenen Verantwortung: Jede Server-Funktion muss `ensureSession()` prüfen, denn `beforeLoad` ist nur UX und keine Sicherheitsgrenze. – [TanStack Start: Authentication](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)
- **PWA-Bequemlichkeit:** Sehr gut. Kein Domainwechsel, lange gleitende Session, Login per Face ID in 1–2 s.
- **Offline:** Neutral bis gut, siehe [Offline-Start](#offline-start).
- **Kosten:** 0 € (Open Source). **Aufwand:** mittel, etwa ½–1 Tag inkl. Login-Screen, Passkey-Registrierung und Owner-Hook.
- **Hosting-unabhängig:** läuft auf Cloudflare, Vercel, Netlify und Node.

### B) Cloudflare Access (Zero Trust) vor der App

- **Kosten:** Free-Plan 0 €, „Teams under 50 users“, Logs 24 h. – [Cloudflare SASE/Zero Trust Plans](https://www.cloudflare.com/plans/sase-zero-trust.md)
- **Einrichtung:** Seit 2026-08 per Klick pro Worker („Protect this Worker behind Access“), für alle Domains inkl. `workers.dev` und Preview-URLs. Im Worker gibt es `ctx.access.getIdentity()` ohne eigene JWT-Prüfung, lokal per `access.dev` in `wrangler.jsonc`. WebSockets werden bei Worker-Level-Access nicht unterstützt (403). – [Cloudflare Workers: Cloudflare Access](https://developers.cloudflare.com/workers/configuration/cloudflare-access/), [Changelog 2026-08-14](https://developers.cloudflare.com/changelog/post/2026-08-14-workers-access/), [workers.dev routing](https://developers.cloudflare.com/workers/configuration/routing/workers-dev/)
- **Nur ein Nutzer:** Allow-Policy auf genau eine E-Mail. Mit One-Time-PIN wird nur an Adressen gesendet, die die Policy erlaubt. Der PIN ist ein **Code** (kein Link) und läuft nach 10 min ab. – [Cloudflare: One-time PIN](https://developers.cloudflare.com/cloudflare-one/integrations/identity-providers/one-time-pin/)
- **Session:** Application- und Global-Session haben Default 24 h, **Maximum 1 Monat**. Bei abgelaufener Session werden AJAX/fetch-Subrequests ohne Login-Aufforderung geblockt. Optional liefert Access stattdessen `401`, dafür muss der Client `X-Requested-With: XMLHttpRequest` senden. – [Cloudflare: Session management](https://developers.cloudflare.com/cloudflare-one/access-controls/access-settings/session-management/)
- **PWA-Probleme:** Manifest, Icons und `sw.js`-Updates werden vom Browser teils ohne bzw. mit eigenen Credentials geholt und scheitern dann an Access. Das lässt sich mit Bypass-Policies für diese Pfade umgehen. – Bypass-Konzept: [Cloudflare Access Policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/); PWA-Fehlerbilder sind **Community**-Berichte: [Node-RED Forum](https://discourse.nodered.org/t/dashboard-2-0-pwa-service-worker-prevents-authentication-redirect-when-session-expires/100574), [SilverBullet Community](https://community.silverbullet.md/t/sync-failures-with-cloudflare/2038). Re-Login heißt Redirect auf `<team>.cloudflareaccess.com`, also Domainwechsel im iOS-Standalone-Fenster (siehe oben).
- **Bewertung:** Sicherheit hoch (die App ist gar nicht öffentlich erreichbar, auch nicht für Exploits gegen eigenen Code). Bequemlichkeit mittel: mindestens monatlicher Re-Login per Mail-Code und PWA-Sonderfälle. Aufwand gering, gilt aber **nur bei Hosting auf Cloudflare**.

### C) Vercel Authentication (Deployment Protection)

- Seit 2026-09-09 kann Vercel Authentication auch Production auf **jedem Plan kostenlos** schützen (Scope „All Deployments“). – [Vercel Changelog](https://vercel.com/changelog/protect-production-deployments-for-free-on-every-plan), [Vercel: Deployment Protection](https://vercel.com/docs/deployment-protection)
- Zugriff haben eingeloggte Vercel-User mit Projektzugriff. Der Login läuft per Redirect zu Vercel. Das Cookie gilt pro URL („Tokens are valid for a single URL“). Eine Session-Laufzeit ist **nicht dokumentiert**. Passwortschutz kostet 20 $/Monat und Projekt (Pro), auf Hobby gibt es ihn nicht. – [Vercel Authentication](https://vercel.com/docs/deployment-protection/methods-to-protect-deployments/vercel-authentication)
- **Bewertung:** 0 €, kein Code. Dieselben PWA- und Redirect-Nachteile wie B, die Session-Dauer ist unbekannt, und es geht nur auf Vercel. Gut für Previews.

### D) Netlify private Projekte

- Auf dem Free-Plan kann ein „private project“ nur der Team Owner sehen, Login per Netlify-Account, gilt auch für Production. Private Projekte empfangen keine Webhooks. Passwortschutz erst ab Pro. – [Netlify: Project visibility](https://docs.netlify.com/manage/security/secure-access-to-sites/project-visibility/), [Netlify: Password protection](https://docs.netlify.com/manage/security/secure-access-to-sites/password-protection/)
- **Bewertung:** wie C. Keine Angaben zur Session-Dauer oder zu PWAs, nur auf Netlify.

### E) GitHub-OAuth (z. B. über Better Auth) mit Allowlist

- Kostenlos. Über Better Auth `socialProviders.github` mit `disableImplicitSignUp` plus Owner-Hook (wie A) auf genau ein Konto beschränkbar. – [Better Auth: Options](https://www.better-auth.com/docs/reference/options)
- **Nachteil PWA:** Ein Redirect-Flow über `github.com` ist im iOS-Standalone-Modus fehleranfällig (Community-Berichte oben). Bis Safari 27.2 kann außerdem das im Callback (Top-Level-Navigation) gesetzte Cookie bei CNAME-Custom-Domains auf 7 Tage gekappt werden ([WebKit 27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/)). Taugt als *Recovery-Weg*, ist als Alltags-Login aber schlechter als ein Passkey.

### F) Magic Link

- Der Link aus der Mail öffnet sich auf iOS in Safari, nicht in der Home-Screen-App. Das Cookie landet im falschen Speicher (**Community**, s. o.). **Nicht geeignet** für die installierte iOS-PWA. Ein Code zum Abtippen (OTP) wäre die PWA-taugliche Variante.

### G) Clerk (Hosted)

- Hobby-Plan: 0 €, aber **keine Passkeys**, Session-Dauer **fest 7 Tage** (anpassbar erst ab Pro), Clerk-Branding nicht entfernbar. – [Clerk Pricing](https://clerk.com/pricing). Von TanStack Start als Option gelistet ([TanStack Start: Authentication](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)).
- **Bewertung:** Passt nicht. Einloggen alle 7 Tage widerspricht „selten re-login“, und es kommt eine externe Abhängigkeit hinzu.

### H) Eigene Session (iron-session), ohne Library

- Der TanStack-Start-Guide zeigt ein eigenes Session-Cookie mit iron-session (`httpOnly`, `sameSite: 'lax'`, `ttl`). „Start does not provide a session API.“ – [TanStack Start: Authentication](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)
- Mit *einem* Passwort aus einer Env-Variable wäre das minimal. WebAuthn selbst zu bauen ist dagegen unnötiger Aufwand, und Sicherheitsdetails (Rate-Limit, Vergleich in konstanter Zeit) liegen dann komplett bei einem selbst. Nur sinnvoll, wenn auf eine DB verzichtet werden soll.

## Übersicht

| Option | Sicherheit | PWA-Bequemlichkeit (iOS/Android) | Max. Session | Offline-Start | Kosten | Aufwand | Hosting-Bindung |
|---|---|---|---|---|---|---|---|
| **A Better Auth + Passkey** | hoch | sehr gut (Face ID, kein Domainwechsel) | frei wählbar, gleitend (Chrome-Cap 400 d) | gut (App-Logik) | 0 € | mittel | keine |
| B Cloudflare Access | sehr hoch (App nicht öffentlich) | mittel (Redirect, Mail-Code, PWA-Bypässe) | 1 Monat | ok, Re-Login-Fälle knifflig | 0 € | gering | Cloudflare |
| C Vercel Authentication | hoch | mittel (Redirect, Vercel-Login) | undokumentiert | wie B | 0 € | gering | Vercel |
| D Netlify private | hoch | mittel | undokumentiert | wie B | 0 € | gering | Netlify |
| E GitHub-OAuth | hoch | mäßig auf iOS (Redirect) | frei wählbar | gut | 0 € | mittel | keine |
| F Magic Link | mittel | schlecht auf iOS | frei wählbar | gut | 0 € (+ Mailversand) | mittel | keine |
| G Clerk Hobby | hoch | schlecht (7 Tage fix, keine Passkeys) | 7 Tage | mittel | 0 € | gering | Clerk |
| H iron-session + Passwort | mittel | gut | frei wählbar | gut | 0 € | gering | keine |

## Offline-Start

Die Auth sitzt am Server, die PWA muss aber offline starten. Konsequenzen für Option A (gelten sinngemäß für alle):

1. **Die App-Shell kommt aus dem Service Worker** und braucht keine Auth. Eine `beforeLoad`-Guard, die *immer* `getSession()` auf dem Server abfragt, würde offline scheitern. Die Client-Guard sollte daher offline nicht hart umleiten: lokale Daten anzeigen und Erfassungen lokal speichern (TanStack DB/IndexedDB). Das Gerät selbst schützt die Gerätesperre.
2. **Die Sicherheitsgrenze sind die Server-Funktionen und Sync-Endpunkte:** Jede prüft `ensureSession()`. Ist die Session beim nächsten Online-Sync abgelaufen → `401` → Login-Screen (Passkey) → Warteschlange abarbeiten. Es geht nichts verloren.
3. Nach dem ersten Login einmal `navigator.storage.persist()` anfragen. Das schützt die Offline-Daten vor Eviction, in der Home-Screen-App wird es eher gewährt ([WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)).
4. Bei B/C/D kommt hinzu: Ein vom SW gecachter Shell-Start umgeht den Plattform-Login, und abgelaufene Sessions zeigen sich erst als geblockte `fetch`-Requests. Man braucht eigens eine „Neu anmelden“-Navigation am SW vorbei und muss Manifest und SW per Bypass freigeben.

## Empfehlung (konkret)

1. **Better Auth** mit `passkey()` und `emailAndPassword` (Passwort nur als Fallback, im Passwortmanager), `tanstackStartCookies()` als letztes Plugin, Auth-Tabellen in der App-DB.
2. **Einzelnutzer:** `databaseHooks.user.create.before` lehnt alles ab, was nicht `OWNER_EMAIL` ist oder wenn schon ein User existiert. Nach dem Einrichten `emailAndPassword.disableSignUp: true` (Sign-up-Pfade ggf. per `disabledPaths` sperren). `OWNER_EMAIL` und `BETTER_AUTH_SECRET` nur als Env/Secrets beim Hoster, **nie in Git**.
3. **Session:** `expiresIn` ≈ 180 Tage, `updateAge` 1 Tag. Auth auf **derselben Origin** wie die App (keine Auth-Subdomain), Cookie HttpOnly/Secure/Lax.
4. **Login-UX:** Ein Button „Mit Passkey anmelden“ (die User-Geste ist auf iOS nötig). Der erste Passkey wird direkt nach dem ersten Passwort-Login *in der installierten App* registriert. Auf iOS ist eine zweite Anmeldung in der Home-Screen-App nötig, weil sie einen eigenen Cookie-Speicher hat.
5. **Optional:** Cloudflare Access bzw. Vercel Authentication nur für **Preview-Deployments** (Standard Protection), nicht für Production.
6. **Vor dem ADR testen:** ein Mini-Prototyp (Passkey-Login in der installierten PWA auf dem eigenen iPhone und Android-Gerät). Der Punkt „WebAuthn im iOS-Standalone-Modus“ ist der einzige nicht durch Primärquellen belegte Baustein.

## Offene Punkte / Abhängigkeiten

- Die Hosting-Entscheidung beeinflusst nur B–D. A ist davon unabhängig. Die DB-Wahl (D1/SQLite/Postgres) kommt aus dem Persistenz-/Sync-Ticket, Better Auth unterstützt alle.
- Passkey-Wiederherstellung: Synced Passkeys (iCloud/Google) plus Passwort-Fallback reichen für einen Einzelnutzer. Ein DB-Zugang zum Notfall-Reset ist die letzte Reserve.
