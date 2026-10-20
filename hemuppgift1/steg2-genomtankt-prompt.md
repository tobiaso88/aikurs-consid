# Hemuppgift 1, Del A, Steg 2: Genomtänkt prompt

**Repo:** https://github.com/lemmynet/lemmy

**Agent** Opus 5.5 medel

**Prompt** Du är en erfaren php/laravel utvecklare som ska sätta dig in i ett Rust-projekt du inte jobbat med tidigare. Analysera kodbasen i detta repo.

**Svar**

# Kodbasanalys – Lemmy backend

> Analys av `LemmyNet/lemmy` (version `1.0.0-beta.2`, commit `fc640cf86`).
> Lemmy är en federerad länkaggregator och forumplattform, ungefär ett självhostat Reddit, som pratar **ActivityPub** med andra servrar i Fediverse.
> Repot innehåller **bara backend/API:t**. Webbgränssnittet (`lemmy-ui`) ligger i ett separat repo.

---

## 1. Snabbfakta

| | |
|---|---|
| Språk | Rust (edition 2024, toolchain ≥ 1.92) |
| Webbramverk | actix-web 4 |
| Databas | PostgreSQL via Diesel 2 + diesel-async |
| Federation | ActivityPub (`activitypub_federation`), WebFinger, NodeInfo |
| Kodstorlek | ~40 crates, ca 90 000 rader Rust, 344 SQL-migreringar, ~1 400 rader SQL-triggers |
| Tester | Enhetstester/DB-tester i Rust + federationstester i TypeScript/Jest (`api_tests/`) |
| CI | Woodpecker (`.woodpecker.yml`) |
| Licens | AGPL-3.0 |

---

## 2. Arkitektur och lager

Kodbasen är en **modulär monolit**. Den kompileras till en enda binär (`lemmy_server`), men koden är uppdelad i ett Cargo-workspace med ett fyrtiotal crates. Beroendena mellan crates styr vilket lager som får anropa vilket, och kompilatorn upprätthåller det.

```
                   ┌─────────────────────────────────────────────┐
                   │  crates/server  (main, bootstrap, wiring)   │
                   └───────────────┬─────────────────────────────┘
        ┌──────────────────────────┼──────────────────────────────┐
        ▼                          ▼                              ▼
┌───────────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐
│ api/routes    │   │ apub/apub (HTTP inbox,   │   │ routes (feeds, bilder,   │
│ api/routes_v3 │   │ collections)             │   │ webfinger, nodeinfo,     │
│ (URL→handler) │   │                          │   │ middleware, cron-jobb)   │
└──────┬────────┘   └────────────┬─────────────┘   └────────────┬─────────────┘
       ▼                         ▼                              │
┌───────────────┐   ┌──────────────────────────┐                │
│ api/api       │   │ apub/activities          │                │
│ api/api_crud  │◄─►│ apub/objects             │                │
│ (use cases)   │   │ apub/send (utkö-workers) │                │
└──────┬────────┘   └────────────┬─────────────┘                │
       ▼                         ▼                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ api/api_utils  – LemmyContext, behörighetskontroller, notifieringar,        │
│                  plugins, HTTP-klient, pict-rs, länkmetadata                │
└──────┬──────────────────────────────────────────────────────────────────────┘
       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ db_views/*  – läsmodeller (joins + API-DTO:er i api.rs), en crate per vy    │
│ db_schema   – entiteter (source/), queries per tabell (impls/)              │
│ db_schema_file – Diesel-genererat schema.rs, enums, newtype-ID:n            │
│ diesel_utils – pool, migreringar, paginering, Crud-trait                    │
└──────┬──────────────────────────────────────────────────────────────────────┘
       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ utils – settings, felhantering, rate limiting, markdown, validering, slurs  │
│ email – SMTP-utskick + översättningar (git-submodul)                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Lagren i detalj

| Lager | Crates | Ansvar |
|---|---|---|
| **Bootstrap** | `crates/server` | Läser CLI-argument (clap), kör migreringar, bygger DB-pool, `LemmyContext`, federation-konfig, startar HTTP-server, schemalagda jobb och federationsutskick. Se [lib.rs](crates/server/src/lib.rs). |
| **Routing** | `api/routes`, `api/routes_v3`, `routes` | Mappar URL:er till handler-funktioner. `routes_v3` är ett kompatibilitetslager för det gamla 0.19-API:t. Det konverterar gamla DTO:er till nya. |
| **Applikation / use cases** | `api/api`, `api/api_crud` | En fil per endpoint, t.ex. [create.rs](crates/api/api_crud/src/post/create.rs). `api_crud` sköter skapa/läsa/uppdatera/ta bort, och `api` sköter övriga actions (rösta, spara, låsa, banna, logga in med mera). |
| **Delade tjänster** | `api/api_utils` | `LemmyContext` (DB-pool, HTTP-klienter, JWT-secret, rate limiter), behörighetskontroller, notifieringar, pluginhooks, bildhantering och länkmetadata. |
| **Federation** | `apub/*` | ActivityPub-objekt (`objects`), aktiviteter som Create, Like, Follow och Delete (`activities`), HTTP-inbox (`apub`) och utgående kö med en worker per mottagande instans (`send`). |
| **Läsmodeller** | `db_views/*` (23 crates) | Komplexa SELECT-frågor med joins, sortering och cursor-paginering. Varje crate har `api.rs` (request/response-DTO:er) och `impls.rs` (queries). |
| **Persistens** | `db_schema`, `db_schema_file`, `diesel_utils` | Entiteter, insert- och update-forms, CRUD per tabell, Diesel-schema och migreringar. |
| **Infrastruktur** | `utils`, `email` | Konfiguration (HJSON), felenum, rate limiting, markdown-rendering, validering och e-post. |

### Request-flöde (exempel: skapa inlägg)

1. `POST /api/v4/post` → `api/routes` → `create_post`.
2. Middleware: CORS, kompression, tracing, idempotens, session/JWT (`SessionMiddleware`), rate limit och Prometheus.
3. `LocalUserView` injiceras som en *extractor*, alltså den inloggade användaren.
4. Handlern kör följande steg:
   - honeypot-kontroll
   - slur-filter
   - URL-blocklist
   - markdown-bearbetning
   - validering
   - behörighetskontroll (ban, moderator-krav)
   - pluginhook *before*
   - `Post::create`
   - pluginhook *after*
   - taggar
   - hämtning av länkmetadata och thumbnail via pict-rs
   - självröstning
   - notifieringar
   - federation via `SendActivityData::CreatePost`
5. Postgres-triggers uppdaterar score, räknare och "combined"-tabeller.
6. Svar byggs via `PostView` från `db_views/post`.

### Driftmodell

`lemmy_server` är en enda binär som kan skalas horisontellt med flaggor:

| Flagga | Effekt |
|---|---|
| `--disable-http-server` | Ingen HTTP-server |
| `--disable-scheduled-tasks` | Inga schemalagda jobb |
| `--disable-activity-sending` | Inga utgående federationsaktiviteter |
| `--federate-process-index` / `--federate-process-count` | Delar upp utgående federation på flera processer |

Därför kan samma binär köras i rollerna API-nod, cron-nod eller federations-worker.

---

## 3. Externa beroenden och vad de gör

### Ramverk och runtime

| Beroende | Roll |
|---|---|
| `tokio` | Asynkron runtime med trådar, timers och kanaler |
| `actix-web` | HTTP-server, routing, middleware, extractors |
| `activitypub_federation` | Lemmy-teamets eget bibliotek för ActivityPub: HTTP-signaturer, inbox-hantering och hämtning/verifiering av remote-objekt |
| `clap` | CLI-argument och miljövariabler |
| `tikv-jemallocator` | Alternativ minnesallokator på x86_64 |

### Databas

| Beroende | Roll |
|---|---|
| `diesel`, `diesel-async` | Typsäker ORM och query builder, async med connection pool via `deadpool` |
| `diesel_migrations` | Kör SQL-migreringar från `migrations/` |
| `diesel_ltree` | Postgres `ltree` för kommentarsträd (`path`-kolumn) |
| `diesel-derive-enum`, `diesel-derive-newtype` | Postgres-enums och typade ID:n (`PostId`, `PersonId` …) |
| `diesel-uplete` | "Update-or-delete" för action-tabeller (t.ex. `post_actions`) |
| `i-love-jesus` | Cursor-baserad (keyset) paginering |
| `tokio-postgres(-rustls)` | TLS-anslutning till Postgres |

### HTTP-klient och nätverk

| Beroende | Roll |
|---|---|
| `reqwest` + `reqwest-middleware` + `reqwest-tracing` | Utgående HTTP mot federation, pict-rs, OAuth och länkmetadata |
| `rustls` | TLS utan OpenSSL |
| `url`, `urlencoding`, `http` | URL-hantering |

### Säkerhet och autentisering

| Beroende | Roll |
|---|---|
| `bcrypt` | Lösenordshashning |
| JWT (via `claims.rs`) | Sessionstoken som valideras mot `login_token`-tabellen |
| TOTP (via `api/local_user/*totp*`) | Tvåfaktorsautentisering |
| `actix-extensible-rate-limit`, `dashmap` | Rate limiting per IP och åtgärdstyp |

### Övrigt

| Beroende | Roll |
|---|---|
| `serde`, `serde_json`, `serde_with`, `deser-hjson` | Serialisering. Konfig skrivs i HJSON. |
| `markdown-it` + plugins (spoiler, sub/sup, ruby, footnote) | Markdown → HTML |
| `clearurls` | Tar bort tracking-parametrar från länkar |
| `moka` | In-memory-cache (instanser, blocklistor, slur-regex, inbox-urval) |
| `clokwerk` | Schemaläggare för cron-liknande jobb |
| `extism` | WebAssembly-pluginsystem (feature-flaggan `plugins`) |
| `lettre` | SMTP för e-post |
| `rosetta-i18n` | Översättningar av e-posttexter |
| `ts-rs` | Genererar TypeScript-typer från Rust-DTO:er åt `lemmy-js-client` |
| `doku` | Genererar dokumenterad `config/defaults.hjson` från Settings-structen |
| `tracing`, `tracing-subscriber`, `tracing-actix-web` | Strukturerad loggning (även JSON) |
| `anyhow`, `strum`, `itertools`, `chrono`, `uuid`, `regex`, `html2text` | Standardverktyg |

---

## 4. Integrationer mot externa system

| System | Var | Beskrivning |
|---|---|---|
| **PostgreSQL** | `diesel_utils/connection.rs` | Primär lagring. Mycket logik ligger i triggers (se avsnitt 5). |
| **Andra Fediverse-servrar** (Lemmy, Mastodon, PieFed, Mbin, Kbin …) | `crates/apub/*` | ActivityPub in/ut. Inbox på `/inbox` och per aktör. Utkö per instans i `apub/send` med persistent position i `federation_queue_state`. HTTP-signaturer och valfri *signed fetch*. Allowlist/blocklist per instans. |
| **WebFinger / NodeInfo** | `routes/webfinger.rs`, `routes/nodeinfo.rs` | Upptäckt av användare och communities samt metadata om instansen. |
| **pict-rs** (separat bildserver) | `routes/images/*`, `api_utils/request.rs` | Uppladdning, proxy, thumbnails, borttagning och purge. Har en egen HTTP-klient utan proxy. |
| **SMTP** | `crates/email` | Verifiering, lösenordsåterställning, notifieringar och admin-mejl. |
| **OAuth2 / OIDC-leverantörer** | `api_crud/user/create.rs` (`authenticate_with_oauth`), `api_crud/oauth_provider/*` | Admin konfigurerar leverantörer i DB. Token endpoint och userinfo endpoint anropas, med valfri PKCE per leverantör. |
| **Godtyckliga webbsidor** | `api_utils/request.rs` | Hämtar OpenGraph-metadata och thumbnails för länkade inlägg, med storleksbegränsning. |
| **Webmention** | `send_webmention` i `api_utils/utils.rs` | Pingar länkade sidor när ett inlägg publiceras. |
| **Bootstrap-instanser** | `setup.bootstrap_instances` | Hämtar populära communities från andra instanser vid första start. |
| **WASM-plugins** | `api_utils/plugins.rs` | Laddas från fil eller URL med SHA256-verifiering. Hooks finns före och efter create/update, för notifieringar, captcha och schemalagda jobb. Plugins kan göra HTTP-anrop mot `allowed_hosts`. |
| **Prometheus** | `routes/utils/prometheus_metrics.rs` | Exponerar mätvärden på separat port. |
| **RSS/Atom** | `routes/feeds` | Publika och privata flöden. |
| **lemmy-translations** | git-submodul i `crates/email/translations` | Översättningar. |

---

## 5. Var affärslogiken finns

Affärslogiken är **utspridd över fyra ställen**, och det är viktigt att känna till alla fyra:

1. **Handler-funktionerna i `api/api` och `api/api_crud`** (mest synligt). Varje fil är en *transaction script*: validering, behörighet, skrivning och sidoeffekter i en sekvens. Det finns ingen separat service- eller domänmodell.
2. **`api/api_utils/src/utils.rs`** (~1 200 rader). Delade regler som `check_community_user_action`, ban-kontroller, NSFW-regler, markdown-bearbetning, blocklistor och purge-logik. Här ligger de närmaste motsvarigheterna till "domain services".
3. **Postgres-triggers** i [triggers.sql](crates/diesel_utils/replaceable_schema/triggers.sql) (~1 100 rader). Triggrarna sköter:
   - score, upvotes/downvotes och controversy rank
   - räknare (antal kommentarer och inlägg, aktiva användare)
   - "combined"-tabeller för rapporter, personinnehåll, sparat och gillat
   - automatiska ändringar av fält vid update (`change_values`-triggers)

   Schemat `r` släpps och återskapas vid varje start ("replaceable schema"), så triggrarna versioneras inte som vanliga migreringar.
4. **Federationslagret** (`apub/activities`, `apub/objects`). Här finns en egen uppsättning regler för vad som är tillåtet när en *remote* aktör gör något: verifiering av moderatorskap, domänkontroller och tombstones. Mycket av logiken dupliceras i förenklad form jämfört med den lokala API-vägen.

Dessutom finns regler i:

- **`utils/src/utils/validation.rs`, `slurs.rs`, `markdown/`**: ren indatavalidering.
- **`routes/src/utils/scheduled_tasks.rs`** (~1 460 rader): hot rank-uppdatering, schemalagda inlägg, utgångna bans och inbjudningar, statistik och rensning.
- **`db_views/*/impls.rs`**: synlighetsregler (privata communities, blockeringar, NSFW, borttaget/removed) är inbakade i SQL-frågorna.

---

## 6. Teknisk skuld och uppenbara problem

### Arkitektur och struktur

- **Logik i databastriggers.** Kraftfullt och snabbt, men logiken syns inte i Rust-koden, är svår att testa och felsöka, och kan ge låskonflikter (filen har själv långa kommentarer om låsordning för att undvika deadlocks).
- **Ingen domänmodell eller servicelager.** Handlers är långa procedurella funktioner. Samma kontrollsekvens (slurs → blocklist → markdown → validering → behörighet) upprepas i create och update för post, comment, community och private message. Risken är att en kontroll glöms i någon variant.
- **Få transaktioner.** Bara ~18 `run_transaction` i hela kodbasen. En operation som `create_post` gör många separata skrivningar utan gemensam transaktion: insert, taggar, like, read-markering och notifieringar. Om något steg misslyckas mitt i blir datan halvfärdig.
- **Utgående aktiviteter går via en obegränsad in-memory-kanal** (`UnboundedSender` i `send_activity.rs`) innan de persisteras. Vid krasch eller hård omstart kan aktiviteter i kanalen förloras, och det finns inget backpressure-tryck.
- **Kraftig crate-fragmentering.** 23 `db_views`-crates med feature-flaggor (`full`, `ts-rs`) ger snabbare inkrementella byggen men lång initial kompilering och mycket boilerplate i `Cargo.toml`.
- **Globalt `SETTINGS`-singleton** (`LazyLock`). Det används direkt på många ställen i stället för via `LemmyContext`, vilket försvårar tester med olika konfiguration.
- **Kompatibilitetslagret `routes_v3`** drar in hela det gamla 0.19-API:t (`lemmy_api_019`) som beroende och har ~900 rader manuell konvertering. Det är nödvändigt för klienter under övergången, men måste underhållas parallellt.

### Konkreta småfel och kodlukt

- **Workspace-listan i `Cargo.toml`** har `crates/db_views/local_image` **två gånger**, medan `registration_applications` **saknas** bland `members`. Den kommer med ändå eftersom den är ett path-beroende, men listan är inkonsekvent.
- **Cirka 60 TODO/FIXME i Rust-koden**, bland annat:
  - `person.rs:598–622`: *"fix person aggregate comment score calculation"*, alltså känt felaktiga aggregat.
  - `local_site.rs:124`: *"this is unstable, sometimes it returns 0 users, sometimes 1"*.
  - `comment/impls.rs:323`: *"a kludge to prevent attacks. Limit comments to 300"*.
  - `modlog/impls.rs:419`: *"why index 2 again?"* i ett test.
  - `markdown/mod.rs`: kända Scunthorpe-problem i slur-filtret.
  - `custom_emoji/impls.rs:29`: *"this type is a mess"*.
- **Stora filer:** `scheduled_tasks.rs` (1 462 rader), `api_utils/utils.rs` (1 215), `notify.rs` (958) och `user/create.rs` (846, med både registrering och hela OAuth-flödet).
- **Exempelkonfigurationen** pekar på ett externt plugin på GitHub och innehåller ett hårdkodat exempel-lösenord för admin. Det är ofarligt i sig men lätt att råka driftsätta.
- **Federationstesterna** körs i ett separat TypeScript-projekt med flera Lemmy-instanser via skript. De är tunga att köra lokalt.

### Det som är bra

- Mycket strikta Clippy-regler: `unwrap_used`, `expect_used`, `indexing_slicing` och `as_conversions` är alla `deny`. Det ger få paniker i runtime.
- Typade ID:n (`PostId`, `PersonId`) förhindrar förväxlingar.
- Cursor-paginering, horisontell skalning och idempotens-middleware.
- Utförliga DB-tester per vy och prestandaskript (`db_perf`, `scripts/query_testing`).

---

## 7. Skillnader mot ett klassiskt PHP/Laravel-projekt

| Område | Laravel | Lemmy (Rust) |
|---|---|---|
| **Körmodell** | Share-nothing: varje request startar om PHP-processen (PHP-FPM). Inget delat minne mellan requests. | En långlivad process med async-runtime (tokio). Delat tillstånd i minnet: cache (moka), rate limiter, DB-pool, kanaler. Bakgrundsjobb körs **i samma process**. |
| **Kompilering och typer** | Dynamiskt typat, fel upptäcks i runtime. | Statiskt typat och kompilerat. SQL-frågor typkontrolleras mot `schema.rs` vid kompilering. Lång byggtid, men många fel stoppas innan de når produktion. |
| **Projektstruktur** | En app med `app/Http/Controllers`, `app/Models`, `app/Services` och konvention före konfiguration. | Ett Cargo-workspace med ~40 crates där lagren separeras via crate-beroenden i stället för mappkonventioner. |
| **Routing** | `routes/api.php` med controllerklasser. | `api/routes/src/lib.rs` med fria funktioner (`create_post`). Inga controllerklasser. |
| **ORM** | Eloquent (Active Record): `$post->save()`, lazy loading, magiska relationer. | Diesel (query builder + data mapper). Separata structs för läsning (`Post`), insert (`PostInsertForm`) och update (`PostUpdateForm`). Ingen lazy loading. Joins skrivs explicit i `db_views`. |
| **Models vs Views** | En modell per tabell som även används för presentation (API Resources). | Entitet (`source/`) skiljs från läsmodell (`db_views/*View`) och från API-DTO (`api.rs`). Ungefär CQRS-light. |
| **Validering** | FormRequest-klasser med deklarativa regler (`'title' => 'required|max:200'`). | Imperativa funktionsanrop i varje handler (`is_valid_post_title(...)?`). |
| **Dependency injection** | Service container med autowiring och facades. | Ingen container. `LemmyContext` skickas via actix `Data<>`, och den inloggade användaren via extractors (`LocalUserView` som funktionsparameter). |
| **Middleware** | `app/Http/Middleware` + Kernel. | actix-middleware i `routes/middleware` (session, idempotens) och `.wrap()` i `server/lib.rs`. |
| **Felhantering** | Exceptions + `Handler.php`. | Inga exceptions. `Result<T, LemmyError>` med `?` och ett centralt enum (`LemmyErrorType`, ~140 varianter) som serialiseras till JSON. |
| **Köer och jobb** | Laravel Queues (Redis/DB) + `php artisan schedule:run` via cron. | Ingen extern kö. Utgående federation använder DB-tabeller (`sent_activity`, `federation_queue_state`) och tokio-workers. Schemalagda jobb körs med `clokwerk` i processen. Inget Redis. |
| **Events och listeners** | Events, Observers, Notifications. | Inga events. Sidoeffekter anropas explicit (`NotifyData::send`, `SendActivityData`). Härledd data uppdateras av Postgres-triggers i stället för Observers. |
| **Migreringar** | PHP-klasser med Schema Builder. | Rå SQL (`up.sql`/`down.sql`) via Diesel. Därtill ett "replaceable schema" med triggers och funktioner som återskapas vid varje start. |
| **Autentisering** | Sanctum/Passport/Breeze. | Egen JWT + `login_token`-tabell, bcrypt, TOTP och OAuth/OIDC. Allt är skrivet i projektet. |
| **Konfiguration** | `.env` + `config/*.php`. | En typad `Settings`-struct som laddas från HJSON och kan överstyras med env-variabler. Dokumentationen genereras automatiskt (`doku`). |
| **Templating/frontend** | Ofta Blade eller Inertia i samma repo. | Ren JSON-API. Frontend ligger i separata repon. TypeScript-typer genereras från Rust (`ts-rs`). |
| **Tester** | PHPUnit/Pest med `RefreshDatabase`. | `#[tokio::test]` + `serial_test` mot riktig Postgres, plus Jest-tester mot flera körande instanser för federation. |
| **Plugins/extensibility** | Composer-paket och service providers. | WebAssembly-plugins (Extism), sandboxade och språkoberoende. |
| **Prestanda/drift** | Skalas med fler FPM-workers. | En binär med inbyggd async-I/O. Skalas med flaggor som fördelar roller (HTTP, cron, federation). |
| **Domänspecifikt** | Typiskt ett centralt system. | **Federerat.** En stor del av koden (`apub/*`) hanterar andra servrar som kan skicka godtycklig eller fientlig data. Det har ingen motsvarighet i ett vanligt Laravel-projekt. |

### Råd för en Laravel-utvecklare som ska in i koden

1. Börja i `crates/api/routes/src/lib.rs` för att hitta endpointen. Följ sedan funktionen till `api/api` eller `api/api_crud`.
2. Data som "räknas ut av sig själv" (score, antal kommentarer) kommer från `crates/diesel_utils/replaceable_schema/triggers.sql`, inte från Rust-koden.
3. Läs `LemmyErrorType` i `crates/utils/src/error.rs` för att förstå vilka fel API:t kan returnera.
4. Allt som har med andra servrar att göra ligger under `crates/apub/`.
