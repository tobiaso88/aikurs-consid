# Hemuppgift 1, Del A, Steg 1: Slarvigt formulerad prompt

**Repo:** https://github.com/lemmynet/lemmy
**Agent** Opus 5.5 medel
**Prompt** Förklara vad det här projektet gör och hur det är uppbyggt.

**Svar**

## Vad projektet är

**Lemmy** är en länkaggregator och diskussionsplattform, ungefär som Reddit, som ingår i **fediversum**. Varje Lemmy-server (en "instans") är fristående med egna användare och communities. Instanserna pratar med varandra över **ActivityPub**, samma protokoll som Mastodon använder. En användare på en instans kan därför följa, rösta och kommentera i communities på andra instanser, och kommunicera med andra ActivityPub-tjänster.

Det här repot innehåller **backend-servern**, skriven i Rust (version `1.0.0-beta.2`, licens AGPL-3.0). Webbgränssnittet `lemmy-ui` ligger i ett eget repo och körs som en separat container i [docker-compose.yml](docker/docker-compose.yml).

## Tekniska grunder

- **Rust 2024**, med webbramverket **actix-web**
- **PostgreSQL** via **Diesel** som ORM, med 344 migrationer i [migrations/](migrations/)
- **activitypub_federation**, ett bibliotek från samma team, för federeringen
- **Extism** för WebAssembly-plugins ([plugins.rs](crates/api/api_utils/src/plugins.rs))
- Konfiguration i HJSON: [config/defaults.hjson](config/defaults.hjson)

## Uppbyggnad

Repot är en Cargo-workspace med många små crates i [crates/](crates/). Uppdelningen gör kompileringen snabbare och håller lagren isär. Från botten och uppåt:

| Lager | Crate(s) | Ansvar |
|---|---|---|
| **Verktyg** | `utils` | Fel, inställningar, rate limiting, hjälpfunktioner |
| **Databas** | `db_schema_file`, `diesel_utils`, `db_schema` | Diesel-schemat, anslutningspool, en Rust-struct för varje tabell (`source/`) med CRUD-implementationer (`impls/`) |
| **Vyer** | `db_views/*` (~25 st) | Frågor som slår ihop flera tabeller till det API:t returnerar, t.ex. `PostView` och `CommentView`. Här finns sortering, filtrering och paginering |
| **E-post** | `email` | Utskick och översättningar |
| **API** | `api/api_common` | Typerna för förfrågningar och svar i API:t |
| | `api/api_utils` | Gemensam logik: kontext, autentisering (claims), notiser, att skicka aktiviteter |
| | `api/api` | Handlers för handlingar som rösta, spara, moderera och rapportera |
| | `api/api_crud` | Handlers som skapar, läser, uppdaterar och tar bort poster, communities, användare m.m. |
| | `api/routes`, `api/routes_v3` | Routing för API v4, plus ett kompatibilitetslager för det äldre v3-API:t ([convert.rs](crates/api/routes_v3/src/convert.rs)) |
| **Federering** | `apub/objects` | Översättning mellan databasobjekt och ActivityPub-objekt (Note, Page, Group, Person…) |
| | `apub/activities` | Inkommande och utgående aktiviteter: Create, Vote, Follow, Delete, Block… |
| | `apub/apub` | ActivityPub-endpoints över HTTP och collections (outbox, followers…) |
| | `apub/send` | Kö och workers för att leverera utgående aktiviteter till andra instanser |
| **Övriga routes** | `routes` | RSS-flöden, bildproxy mot pict-rs, webfinger, nodeinfo, middleware |
| **Server** | `server` | Själva programmet: startar allt |

### Hur det hänger ihop

[crates/server/src/lib.rs](crates/server/src/lib.rs) innehåller `start_lemmy_server`, som:

1. läser in konfigurationen och bygger databaspoolen,
2. startar plugins i bakgrunden,
3. konfigurerar federeringen, om den är påslagen för instansen,
4. startar de schemalagda jobben och `SendManager` för utgående federering,
