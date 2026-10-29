# Research: Kassenbon-Auslesen per KI

Ticket: [#5](https://github.com/ExziiL/snack_v5/issues/5) (Kind von Map-Issue #1). Stand: 2026-10-30.
Alle Preise in USD laut Anbieter-Preisseiten (USD ≈ EUR für diese Größenordnung). Kostenrechnungen sind **Schätzungen** mit den unten genannten Annahmen.

## TL;DR

- **Alle drei großen Anbieter (Anthropic, OpenAI, Google) können das technisch:** Vision-Input + JSON-Schema-erzwungene Ausgabe. Ein Bon kostet je nach Modell **0,1 bis 4,5 Cent**. Selbst das teuerste sinnvolle Modell bleibt bei ~60 Bons/Monat unter 3 €.
- **Die Evidenz zur Genauigkeit bei deutschen Bons ist dünn.** Es gibt keinen öffentlichen Benchmark auf deutschen Supermarkt-Bons. Der beste aktuelle Receipt-Benchmark (ReceiptBench, ACL 2026) zeigt: Kopffelder (Laden, Datum, Summe) klappen gut, **Positionen (Line Items) sind überall die Schwachstelle** (F1 0,49 bis 0,65). Außerdem „schönen“ Modelle manchmal Preise, damit die Summe aufgeht.
- **Empfehlung:** `claude-sonnet-5-5` über die Anthropic Messages API mit Structured Outputs (`output_config.format` + Zod), aufgerufen aus einer TanStack-Start-Server-Function, Bild clientseitig verkleinert und inline als Base64 geschickt (nicht über die Files API, sonst wird es gespeichert). Dazu eine Summenprüfung im Server und eine UI, in der jede Position bestätigt wird. **Vor der Festlegung eine Mini-Eval mit 15 bis 20 eigenen Bons** (Haiku 5.5 vs. Sonnet 5.5, optional Gemini Flash). Ist Haiku dort gleich gut, auf Haiku wechseln (~0,1 Cent/Bon).
- Spezialisierte Receipt-OCR-APIs (Azure Document Intelligence, Mindee, Veryfi) sind eine Alternative, aber starrer (fixes Schema, kein Pfand/Rabatt-Verständnis per Prompt) und teils mit Mindestgebühren. Für dieses Projekt nicht nötig.

---

## 1. Geeignete Modelle (Stand Oktober 2026)

### Anthropic (Claude)

| Modell | Model ID | Input / Output $ pro MTok | Vision-Tier |
|---|---|---|---|
| Claude Haiku 5.5 | `claude-haiku-5-5` | $0.10 / $0.50 (Prompt ≤ 100K Tokens) | High-Res (Claude 4.7+) |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | $2 / $10 | High-Res |
| Claude Opus 5.5 | `claude-opus-5-5` | $4 / $20 | High-Res |

- Preise: [Anthropic Pricing](https://platform.claude.com/docs/en/about-claude/pricing). Batch-API: 50 % Rabatt (für interaktives Bon-Scannen nicht sinnvoll).
- Alle drei unterstützen Vision und Structured Outputs ([Vision](https://platform.claude.com/docs/en/build-with-claude/vision), [Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)).
- Formate: JPEG, PNG, GIF, WebP. **Kein HEIC**, also iPhone-Fotos clientseitig konvertieren ([Vision: Supported formats](https://platform.claude.com/docs/en/build-with-claude/vision#supported-formats)).
- Hinweise zur API: Opus 5.5 kann Thinking nicht deaktivieren (Steuerung nur über `output_config.effort`, Default `medium`). Sonnet 5.5 und Opus 5.5 lehnen erzwungenes `tool_choice` (`any`/`tool`) ab. Für JSON-Extraktion deshalb **Structured Outputs statt Tool-Use** verwenden. Haiku 5.5 kann mit `stop_reason: "refusal"` ablehnen und hat keinen serverseitigen Fallback.

### OpenAI

Aktuelle Modelle laut [Images & Vision Guide](https://developers.openai.com/api/docs/guides/images-vision) und [Pricing](https://developers.openai.com/api/docs/pricing) (Standard-Tier, ≤ 272K Kontext):

| Modell | Input / Output $ pro MTok |
|---|---|
| `gpt-5.6-luna` | $0.20 / $1.20 |
| `gpt-5.4-mini` | $0.75 / $4.50 |
| `gpt-5.6-terra` | $2.00 / $12.00 |
| `gpt-5.6-sol` | $4.00 / $20.00 |
| `gpt-6-astra` (Flaggschiff) | $10.00 / $50.00 |

- Batch/Flex: etwa 50 % günstiger. Regionale Endpoints (Data Residency): +10 % für Modelle ab 5. März 2026 ([Pricing](https://developers.openai.com/api/docs/pricing)).
- Formate: PNG, JPEG, WEBP, nicht animiertes GIF.
- OpenAI nennt als Grenzen selbst: kleine Schrift, gedrehter Text, Genauigkeit „in some scenarios“ ([Images & Vision: Limitations](https://developers.openai.com/api/docs/guides/images-vision)).

### Google Gemini

Laut [Gemini API Pricing](https://ai.google.dev/gemini-api/docs/pricing) (Paid Tier, Standard):

| Modell | Input / Output $ pro MTok |
|---|---|
| Gemini 3.1 Flash-Lite | $0.25 / $1.50 |
| Gemini 3.5 Flash-Lite | $0.30 / $2.50 |
| Gemini 3.6 Flash / 3.8 Flash | $0.75 / $3.75 bis 31.12.2026, **ab 01.01.2027 $1.50 / $7.50** |
| Gemini 3.1 Pro Preview | $2.00 / $12.00 |
| Gemini 4 Argon (Flaggschiff) | $2.00 / $10.00 bis 30.11.2026, danach teurer |

- Es gibt einen Free Tier (Rate-limitiert, siehe Datenschutz unten).
- Formate: PNG, JPEG, WEBP, **HEIC, HEIF** ([Image understanding](https://ai.google.dev/gemini-api/docs/image-understanding)).

### Spezialisierte Receipt-OCR-APIs (kurz)

| Dienst | Eckdaten | Bewertung für Snack |
|---|---|---|
| **Azure Document Intelligence** `prebuilt-receipt` (v4.0 GA) | Felder u. a. `MerchantName`, `TransactionDate`, `Items` (Description, Quantity, Price, TotalPrice), `Total`. Deutsch ist für Thermo-Bons unterstützt. Free Tier F0: 500 Seiten/Monat. Mindesttexthöhe 12 px bei 1024×768. ([Receipt model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/receipt), [Language support](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/language-support/prebuilt), [Pricing](https://azure.microsoft.com/en-us/pricing/details/ai-document-intelligence/)) | Solide, EU-Region wählbar. Aber fixes Schema: Pfand, Rabatt-Zuordnung und lesbare Namen müssten selbst nachgebaut werden. Azure-Setup ist für ein privates Projekt aufwendig. |
| **Mindee** (Receipt-Modell) | Starter 44/Monat für 6.000 Credits, 14 Tage Trial, kein dauerhafter Free Plan. „Data processing localization“ erst ab Pro. ([Pricing](https://www.mindee.com/pricing)) | Zu teuer für < 5 €/Monat. |
| **Veryfi** | Free: 100 Dokumente/Monat. Starter $0.08/Receipt bei **$500/Monat Mindestumsatz**. ([Pricing](https://www.veryfi.com/pricing/)) | Free Tier würde reichen, ist aber ein US-Anbieter mit fixem Schema. Kein Vorteil gegenüber LLM. |

Fazit: Ein LLM mit eigenem Schema ist hier flexibler (Pfand, Rabatte, Namen ausschreiben, Kategorie vorschlagen) und gleich günstig oder günstiger.

---

## 2. Strukturierte Ausgabe: Mechanismen

| Anbieter | Mechanismus | Garantie | Einschränkungen |
|---|---|---|---|
| Anthropic | `output_config: { format: ... }` (JSON outputs). TS-SDK: `client.messages.parse()` + `zodOutputFormat(schema)` aus `@anthropic-ai/sdk/helpers/zod`. Alternativ Strict Tool Use (`strict: true`). | Antwort validiert gegen das Schema | `additionalProperties: false` überall Pflicht. Nicht unterstützt: `minimum`/`maximum`, `minLength`/`maxLength`, rekursive Schemas. Das SDK entfernt diese und prüft sie clientseitig. Unterstützt: `enum`, `anyOf`, `format: "date"`. Bei `stop_reason` `refusal` oder `max_tokens` kann das Schema verletzt sein. Neue Schemas haben einmalige Compile-Latenz (24 h Cache). ([Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)) |
| OpenAI | Responses API: `text.format = { type: "json_schema", strict: true, schema }`. Chat Completions: `response_format`. SDK-Helfer für Zod/Pydantic. | „always generate responses that adhere to your supplied JSON Schema“ | Alle Felder `required`, `additionalProperties: false`. Optional = `["string","null"]`. Refusal kommt als eigener `refusal`-Content. ([Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)) |
| Google | `response_format` mit `mime_type: "application/json"` + `schema` (Zod/Pydantic im SDK) | „syntactically correct JSON“. Google sagt selbst: „always validate values in your application“ | Einige Constraints werden ignoriert (`minLength`, `uniqueItems` …). Zu große oder tiefe Schemas führen zu `400 INVALID_ARGUMENT`. ([Structured output](https://ai.google.dev/gemini-api/docs/structured-output)) |

**Wichtig:** Schema-Konformität ≠ inhaltliche Korrektheit. Die Summenprüfung und die Bestätigung durch den Nutzer bleiben Pflicht.

---

## 3. Zuverlässigkeit bei deutschen Bons: Evidenz

### Was es an Belegen gibt

- **ReceiptBench** (ACL 2026, [arXiv 2605.22413](https://www.alphaxiv.org/abs/2605.22413)): 10.656 Belege, 19 Felder. **98 % Englisch, nur 31 deutsche Belege.** Ergebnisse als F1:

  | Modell | Gesamt | Line Items („Structure“) |
  |---|---|---|
  | Gemini-3-Pro | 0,737 | 0,578 |
  | GPT-5 | 0,708 | 0,489 |
  | Qwen3-VL-8B (feinjustiert) | 0,795 | 0,637 |

  Bei **jedem** Modell sind die Line Items die schwächste Kategorie. Dokumentierte Fehlerbilder:
  - **„Value Tampering“:** Ein Modell ändert einen Positionspreis oder erfindet eine Steuerposition, damit die Summe aufgeht.
  - **„Unwanted Calculation“:** Fehlt die Summe, rechnet das Modell sie aus, statt `null` zu liefern.
  - Halluzinierte Kopffelder (bei Gemini häufiger), vertauschte Tag/Monat-Angaben bei Datumsformaten wie „02/03/24“, verwechselbare Ziffern (1/7).

  Claude-Modelle, Flash-Modelle und GPT-4o wurden **nicht** getestet. Pfand- oder Rabattzeilen werden nicht behandelt.
- Die Anbieter selbst nennen Schwächen bei **kleinen, gedrehten und unscharfen Bildern** ([Claude Vision: Limitations](https://platform.claude.com/docs/en/build-with-claude/vision#limitations), [OpenAI: Limitations](https://developers.openai.com/api/docs/guides/images-vision)). Google empfiehlt korrekt gedrehte, scharfe Bilder ([Gemini Image understanding](https://ai.google.dev/gemini-api/docs/image-understanding)).
- Einen **unabhängigen Vergleich GPT vs. Claude vs. Gemini auf deutschen Kassenbons (mit Pfand/Rabatt) habe ich nicht gefunden.** Treffer in der Websuche sind Hersteller-Blogs (Klippa, Parseur u. a.) ohne nachvollziehbare Methodik. Sie taugen nicht als Beleg.

### Einschätzung für die typischen Problemfälle

Die folgenden Punkte sind **Erfahrungswerte bzw. Ableitungen, nicht durch Quellen belegt**. Genau deshalb ist die Mini-Eval (Abschnitt 7) wichtig.

| Problemfall | Erwartung | Gegenmaßnahme |
|---|---|---|
| Abgekürzte Namen („SCH.SAHNE 30%“, „BIO H-MILCH 1,5“) | Lesen klappt meist gut, das „Ausschreiben“ ist eine plausible Vermutung des Modells | `rawText` (wörtlich) **und** `name` (ausgeschrieben) getrennt ausgeben. Die UI zeigt beides, der Nutzer korrigiert. |
| Mengenzeilen „2 x 1,29“ bzw. „2 Stk x 1,29“ in eigener Zeile über/unter dem Artikel | Häufigste Fehlerquelle: Menge wird als eigene Position gelesen oder einer falschen Position zugeordnet | Im Prompt explizit beschreiben. Schema mit `quantity`, `unitPriceCents`, `totalPriceCents`. Server prüft `quantity × unitPrice ≈ totalPrice`. |
| Wiegeartikel „0,456 kg x 2,99 EUR/kg“ | Wie oben | `quantityUnit: "kg"` und dezimale `quantity` erlauben |
| Pfand („Pfand 0,25“) und Leergut-Rückgabe (negativ) | Wird gelesen, aber die Zuordnung zum Getränk ist unsicher | Eigener `kind` (`deposit`, `depositReturn`) + `appliesToIndex`. In der UI Pfand standardmäßig nicht als Snack werten, aber dem Getränk zuschlagen können. |
| Rabatte („Rabatt -0,50“, „Preisvorteil“, App-Rabatte) | Negative Zeilen werden meist erkannt. Ob sie zum Artikel davor gehören, ist unsicher. | `kind: "discount"`, `totalPriceCents` negativ, `appliesToIndex` |
| MwSt.-Kennbuchstaben (A/B am Zeilenende), Zwischensummen, Zahlungsart, TSE-Daten | Werden manchmal als Positionen ausgegeben | Im Prompt ausschließen, Zwischensummen nicht als Position |
| Verblasstes Thermopapier, Knicke, Schatten | Größtes Risiko für Ziffernfehler. Keine belastbaren Zahlen. | Gute Aufnahme (Abschnitt 5). `confidence` pro Position. `warnings`-Array. Bei Summenabweichung Warnung anzeigen. |
| Sehr lange Bons | Bei extremem Seitenverhältnis wird das Bild herunterskaliert, die Schrift wird winzig | Bild auf den Bon zuschneiden. Ab etwa 1:3 in 2 überlappende Teilbilder aufteilen (siehe Abschnitt 5). |
| Restaurant-Rechnung (Trinkgeld handschriftlich, Bewirtungsbeleg) | Gedruckte Positionen meist ok. Handschrift unsicher. | `kind: "tip"`, Nutzer bestätigt |

---

## 4. Kosten pro Bon

### Bild-Tokens

- **Claude:** `⌈width/28⌉ × ⌈height/28⌉` visuelle Tokens. High-Res-Tier (Claude 4.7 und neuer, also auch Haiku/Sonnet/Opus 5.5): max. 2.576 px lange Kante bzw. 4.784 Tokens, größere Bilder werden herunterskaliert. Beispiel aus der Doku: 1000×1000 = 1.296 Tokens ([Vision: Resolution and token cost](https://platform.claude.com/docs/en/build-with-claude/vision#evaluate-image-size)).
- **OpenAI (patch-basierte Modelle wie gpt-5.x):** `⌈w/32⌉ × ⌈h/32⌉` Patches × Modell-Multiplikator (1,2 bei gpt-5.6-*, gpt-5.4-*). `detail`: `low`/`high`/`original`/`auto` ([Images & Vision](https://developers.openai.com/api/docs/guides/images-vision)).
- **Gemini 3/4:** über `media_resolution` fix: `low` 280, `medium` 560, `high` 1.120 (Default), `ultra_high` 2.240 Tokens pro Bild. Laut Google gilt für Dokumente/OCR: „quality typically saturates at `medium`“ ([Media resolution](https://ai.google.dev/gemini-api/docs/media-resolution)). Für lange, eng bedruckte Bons würde ich trotzdem `high` testen.

### Schätzung

Annahmen: Foto auf 1200×1600 px verkleinert. ~1.000 Text-Tokens (Prompt + Schema). ~1.500 Output-Tokens (JSON für ~20 Positionen inkl. Thinking bei niedriger Effort-Stufe; der Thinking-Anteil ist die größte Unbekannte).

| Modell | Bild-Tokens | ≈ Kosten/Bon | ≈ 60 Bons/Monat |
|---|---|---|---|
| `claude-haiku-5-5` | 2.494 | **$0.0011** | $0.07 |
| `claude-sonnet-5-5` | 2.494 | **$0.022** | $1.30 |
| `claude-opus-5-5` | 2.494 | $0.044 | $2.60 |
| `gpt-5.6-luna` | ~2.280 | $0.0025 | $0.15 |
| `gpt-5.4-mini` | ~2.280 | $0.009 | $0.55 |
| `gpt-5.6-terra` | ~2.280 | $0.025 | $1.50 |
| Gemini 3.1 Flash-Lite (`high`) | 1.120 | $0.0028 | $0.17 |
| Gemini 3.6/3.8 Flash (`high`) | 1.120 | $0.0072 (ab 2027 ~$0.014) | $0.43 (ab 2027 ~$0.85) |
| Gemini 3.1 Pro Preview (`high`) | 1.120 | $0.022 | $1.30 |
| Veryfi / Azure DI | – | Free Tier 100 bzw. 500 Dokumente/Monat | $0 im Free Tier |

→ **Das Budget von < 5 €/Monat ist mit jedem Modell außer den Flaggschiffen (gpt-6-astra, Fable) eingehalten.** Kosten sind kein Auswahlkriterium, Genauigkeit und Datenschutz schon. Echte Werte nach dem Bau über `usage` in der API-Antwort bzw. `messages.count_tokens` messen.

---

## 5. Bildgröße und Vorverarbeitung (clientseitig in der PWA)

1. **Auf den Bon zuschneiden.** Hintergrund kostet Tokens und Auflösung. Einfacher Weg: Kamera-Overlay mit Rahmen. Optional später automatische Kantenerkennung.
2. **Verkleinern auf ≤ 2.576 px lange Kante** (Claude High-Res-Cap). Größere Bilder skaliert die API sowieso herunter, sie kosten nur Upload-Zeit. Auf 1.568 px muss man nicht runter, die Pixel werden auf High-Res-Modellen tatsächlich genutzt.
3. **Lange Bons:** Das Token-Cap (4.784 × 28² ≈ 3,75 MP) begrenzt bei schmalen Bildern die Breite. Bei 1:4 bleiben ~644 px Breite für 80 mm Bonpapier, das ist grenzwertig. **Ab etwa 1:3 in 2 Teilbilder mit Überlappung teilen** und beide im selben Request schicken (mehrere `image`-Blöcke). Dem Prompt sagen, dass Überlappungszeilen nur einmal zählen.
4. **EXIF-Orientierung anwenden** (z. B. `createImageBitmap(file, { imageOrientation: "from-image" })` → Canvas). Gedrehte Bilder sind laut allen Anbietern eine Fehlerquelle.
5. **JPEG oder WebP, Qualität ~0,85, nur einmal komprimieren.** Anthropic warnt, dass starke oder mehrfache JPEG-Kompression Text unlesbar macht ([Vision: Image quality guidance](https://platform.claude.com/docs/en/build-with-claude/vision)). HEIC vor dem Upload konvertieren (Claude/OpenAI unterstützen es nicht).
6. **Größenlimits:** Claude API 10 MB pro Bild (Base64). Gemini 20 MB pro Request inline. Nach Schritt 2 liegt man typischerweise bei 0,3 bis 1 MB.
7. **Inline Base64 statt Files API.** Die Anthropic Files API speichert Dateien, bis sie gelöscht werden. Das widerspricht „Fotos nicht dauerhaft speichern“.

---

## 6. Datenschutz und Datenhaltung der Anbieter

| | Anthropic (Claude API) | OpenAI API | Google Gemini API |
|---|---|---|---|
| Training auf API-Daten | Nein: „Retained data is never used for model training without your express permission.“ ([API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention)) | Nein, außer bei explizitem Opt-in ([Your data](https://developers.openai.com/api/docs/guides/your-data)) | **Paid:** nein. **Free/Unpaid:** ja, inkl. menschlicher Reviewer. **Aber:** für Nutzer in EWR/CH/UK gelten die Paid-Bedingungen auch für den Free Tier ([Gemini API Terms](https://ai.google.dev/gemini-api/terms)) |
| Standard-Speicherung | Docs: „Conversation content … is not retained by default“ (Ausnahme: „Covered Models“ Fable/Mythos mit 30 Tagen Pflicht). Das Privacy Center nennt dagegen „delete inputs and outputs … within 30 days“. Bei Policy-Verstoß bis 2 Jahre, bei Feedback 5 Jahre ([Commercial retention](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)). **Die beiden Quellen widersprechen sich leicht. Konservativ mit „bis 30 Tage“ rechnen.** | Abuse-Monitoring-Logs bis 30 Tage. Responses API mit `store` (Default) mindestens 30 Tage, also `store: false` setzen. CSAM-Verdacht: Bild wird immer zur Prüfung gespeichert. | Paid: Logs „for a limited period of time“ zur Missbrauchserkennung, ohne Angabe von Tagen |
| ZDR | Auf Anfrage über Sales, pro Organisation | Nur mit vorheriger Genehmigung durch OpenAI | – |
| EU-Option | **Keine** im First-Party-API: `inference_geo` nur `global`/`us`, Workspace-Geo nur `us` ([Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency)). EU nur über Partner (Amazon Bedrock, Google Vertex AI) | `eu.api.openai.com`, setzt aber ZDR/Modified Abuse Monitoring o. ä. voraus (Genehmigung durch Sales) und kostet +10 % | Gemini API: keine Regionswahl dokumentiert. EU-Regionen über Vertex AI (anderes Produkt, GCP-Setup) |

Einordnung für Snack: Für eine private Single-User-App mit Bons (Einkaufsdaten, eventuell letzte Kartenziffern, kein Name) ist **„kein Training + kurze Speicherung“** das relevante Kriterium. Das erfüllen alle drei im bezahlten Modus. EU-Residency wäre nur mit deutlich mehr Setup zu haben und ist für den Zweck unverhältnismäßig. Bei Gemini den Paid Tier nutzen: Die Terms erlauben in EWR/CH/UK beim „making API Clients available to users“ nur Paid Services, und der Free Tier hat Rate Limits. Der API-Key bleibt serverseitig in der TanStack-Start-Server-Function. Anthropic unterstützt bei ZDR ohnehin kein CORS, ein Backend-Proxy ist also sowieso der Weg.

---

## 7. Empfehlung

1. **Provider/Modell:** Anthropic, Start mit **`claude-sonnet-5-5`**, `output_config.effort: "low"` (bei schlechten Ergebnissen `medium`).
   - Begründung: Datenschutz ist ohne Sonderverträge klar (kein Training, keine Retention per Default laut Docs). Structured Outputs mit Zod im TS-SDK. ~2 Cent/Bon.
   - Ehrlicher Vorbehalt: **Es gibt keine harte Evidenz, dass Claude bei Bons besser ist als Gemini oder OpenAI.** Die einzige Benchmark-Evidenz (ReceiptBench) hat Claude gar nicht getestet. Die Wahl beruht auf Datenschutz, Developer Experience und Kosten, nicht auf gemessener Genauigkeit.
2. **Mini-Eval vor der Festlegung** (einmalig, Kosten < 1 €): 15 bis 20 eigene Bons (Aldi, Lidl, Rewe, Edeka, dm, Bäcker, Tankstelle, 2 bis 3 Restaurantrechnungen, mindestens ein verblasster und ein langer Bon). Von Hand Ground Truth anlegen. `claude-haiku-5-5` vs. `claude-sonnet-5-5` vergleichen, optional Gemini 3.x Flash. Metriken: Summe korrekt, Datum korrekt, Positionen korrekt (Preis exakt, Menge exakt). **Ist Haiku 5.5 gleich gut, auf Haiku wechseln** (Faktor ~20 günstiger, schneller).
3. **Pipeline:** PWA (Kamera → Zuschneiden → EXIF → Resize ≤ 2.576 px → JPEG 0,85) → Server Function → `client.messages.parse({ model, output_config: { format: zodOutputFormat(ReceiptSchema), effort: "low" }, messages: [{ role: "user", content: [image (base64), text (Prompt)] }] })` → Validierung → Review-UI. Bild nach dem Request verwerfen. Für Sonnet 5.5 den serverseitigen Refusal-Fallback (`fallbacks: "default"`, Beta `server-side-fallback-2026-07-01`) einschalten. Immer `stop_reason` prüfen.
4. **Prompt-Regeln** (Kern):
   - „Gib Werte exakt wie gedruckt wieder. **Rechne nichts aus und passe keine Werte an, damit die Summe stimmt.** Unlesbares als `null` + `warnings`.“ (gegen „Value Tampering“ und „Unwanted Calculation“)
   - Deutsche Bon-Konventionen beschreiben: Mengenzeile „2 x 1,29“ gehört zur benachbarten Artikelzeile. Wiegeartikel. Pfand/Leergut. Rabattzeilen negativ und auf den Artikel davor bezogen. MwSt.-Kennbuchstaben ignorieren. Zwischensumme, Zahlungsart, TSE und Steuerblock sind keine Positionen.
   - Datum ist im deutschen Format TT.MM.JJ(JJ) und wird als ISO `YYYY-MM-DD` ausgegeben.
5. **Server-Validierung (nicht dem Modell überlassen):** `Σ totalPriceCents == totalCents`, `quantity × unitPriceCents ≈ totalPriceCents` (±1 ct), Datum nicht in der Zukunft. Bei Abweichung: gelbe Warnung in der UI und optional **ein** Retry mit `claude-opus-5-5`. Kein stilles Korrigieren.
6. **UX:** Die KI macht nur einen Vorschlag. Der Nutzer wählt Snack-Positionen per Checkbox und kann Name, Preis und Menge editieren. `rawText` wird klein darunter angezeigt. Die KI kann eine Kategorie vorschlagen (eigenes Feld, mit der Kategorieliste der App als `enum` im Prompt/Schema). Das ist aber optional und kommt erst später.

### Schema-Skizze (Zod → `zodOutputFormat`)

Beträge als **Integer-Cent**, um Float-Fehler zu vermeiden. Keine `min`/`max`-Constraints im Schema, weil Claude Structured Outputs sie nicht unterstützen. Solche Prüfungen gehören in die Server-Validierung.

```ts
import { z } from "zod";

const LineItemKind = z.enum([
  "product",       // normaler Artikel
  "deposit",       // Pfand
  "depositReturn", // Leergut-Rückgabe (negativ)
  "discount",      // Rabatt / Preisvorteil (negativ)
  "tip",           // Trinkgeld
  "serviceCharge",
  "other",
]);

const LineItem = z.object({
  rawText: z.string(),                       // exakt wie gedruckt, ggf. inkl. Mengenzeile
  name: z.string(),                          // lesbar ausgeschrieben ("Schlagsahne 30%")
  kind: LineItemKind,
  quantity: z.number(),                      // 1, 2, 0.456
  quantityUnit: z.enum(["piece", "kg", "l"]),
  unitPriceCents: z.number().int().nullable(),
  totalPriceCents: z.number().int(),         // negativ bei discount / depositReturn
  appliesToIndex: z.number().int().nullable(), // Index der Position, auf die sich Pfand/Rabatt bezieht
  confidence: z.enum(["high", "medium", "low"]),
});

export const ReceiptSchema = z.object({
  isReceipt: z.boolean(),                    // false, wenn kein Bon erkennbar
  documentType: z.enum(["supermarket", "restaurant", "other"]),
  merchantName: z.string().nullable(),       // -> Snack.Laden
  merchantAddress: z.string().nullable(),
  purchaseDate: z.string().nullable(),       // ISO YYYY-MM-DD -> Snack.Datum
  purchaseTime: z.string().nullable(),       // HH:MM
  currency: z.enum(["EUR", "other"]),        // Snack ist EUR-only; "other" -> Warnung
  lineItems: z.array(LineItem),
  totalCents: z.number().int().nullable(),   // gedruckte Endsumme, NICHT berechnet
  warnings: z.array(z.string()),             // z. B. "Zeile 7 unleserlich"
});
```

Zuordnung zum Domänenmodell: Jede vom Nutzer gewählte `product`-Position wird ein Snack. `name` → Name, `totalPriceCents` (+ zugeordnete `discount`/`deposit` nach Nutzerwahl) → Preis, `quantity` → Menge, `merchantName` → Laden, `purchaseDate` → Datum. Die Kategorie wählt der Nutzer (oder sie kommt später als KI-Vorschlag).

---

## Offene Punkte / nicht verifiziert

- Ob Haiku 5.5 für Bilder exakt im High-Res-Tier (2.576 px / 4.784 Tokens) läuft, leite ich aus „Claude 4.7 and later models“ in der Vision-Doku ab. Mit `count_tokens` an einem echten Bild prüfen.
- Thinking-Token-Verbrauch bei `effort: "low"` ist geschätzt. Echte `usage`-Werte in der Eval loggen.
- Widerspruch Anthropic-Docs („not retained by default“) vs. Privacy Center („within 30 days“) ist nicht aufgelöst.
- Exakter Azure-DI-Preis pro 1.000 Seiten war auf der Preisseite nicht lesbar (dynamisch geladen).
- Kein Benchmark für deutsche Supermarkt-Bons gefunden. Die Mini-Eval ist die einzige belastbare Quelle, die wir bekommen werden.

## Quellen

- Anthropic: [Vision](https://platform.claude.com/docs/en/build-with-claude/vision) · [Pricing](https://platform.claude.com/docs/en/about-claude/pricing) · [Structured Outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) · [API and data retention](https://platform.claude.com/docs/en/manage-claude/api-and-data-retention) · [Data residency](https://platform.claude.com/docs/en/manage-claude/data-residency) · [Commercial data retention (Privacy Center)](https://privacy.claude.com/en/articles/7996866-how-long-do-you-store-my-organization-s-data)
- OpenAI: [Images & Vision](https://developers.openai.com/api/docs/guides/images-vision) · [Pricing](https://developers.openai.com/api/docs/pricing) · [Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs) · [Your data](https://developers.openai.com/api/docs/guides/your-data)
- Google: [Gemini API Pricing](https://ai.google.dev/gemini-api/docs/pricing) · [Image understanding](https://ai.google.dev/gemini-api/docs/image-understanding) · [Media resolution](https://ai.google.dev/gemini-api/docs/media-resolution) · [Structured output](https://ai.google.dev/gemini-api/docs/structured-output) · [Gemini API Additional Terms](https://ai.google.dev/gemini-api/terms)
- Receipt-APIs: [Azure DI Receipt model](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/prebuilt/receipt) · [Azure DI language support](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/language-support/prebuilt) · [Azure DI Pricing](https://azure.microsoft.com/en-us/pricing/details/ai-document-intelligence/) · [Mindee Pricing](https://www.mindee.com/pricing) · [Veryfi Pricing](https://www.veryfi.com/pricing/)
- Benchmark: [ReceiptBench – From Recognition to Reasoning: Benchmarking and Enhancing MLLMs on Real-World Receipt Document Understanding (ACL 2026, arXiv 2605.22413)](https://www.alphaxiv.org/abs/2605.22413)
