# Lemmy – kort intro

Repo: https://github.com/LemmyNet/lemmy

## Vad
Typ ett självhostat Reddit. Servrarna pratar med varandra via ActivityPub (som Mastodon), så man kan följa communities på andra instanser.

För folk som vill köra eget forum och de som använder dem.

Repot är bara backenden i Rust, frontend ligger i ett annat repo.

## Arkitektur
En binär men uppdelad i ca 40 crates i lager:

- routes tar emot anrop
- api har en fil per endpoint
- apub sköter federeringen
- db_schema/db_views sköter databasen

Samma binär kan köras som API, cron eller federations-worker med flaggor.

## Tre beroenden
- actix-web – webbramverket
- Diesel + Postgres – typsäkra queries, och Postgres är enda lagringen (ingen Redis eller kö)
- activitypub_federation – deras eget lib för federeringen

## Förvånade mig
Mycket logik ligger i Postgres-triggers. Score, antal kommentarer osv räknas ut i en SQL-fil på ca 1100 rader.

## Agenten
Båda svaren var överens om vad Lemmy är och hur det hänger ihop, så grunderna satt.

Med en slarvig prompt fick jag en okej översikt men inte mer. När prompten var tydligare kom det mycket mer, som triggers, var logiken ligger och teknisk skuld.

Ibland gick svaren isär, t.ex. ca 25 db_views-crates i det ena och 23 i det andra, så alla siffror stämmer inte.
