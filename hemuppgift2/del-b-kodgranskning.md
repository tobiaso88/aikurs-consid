# Hemuppgift 2, Del B - Kodgranskning och säkerhet utan implementation

## Promt 1
```Granska kodbasen mot de arkitekturprinciper jag definierat i CLAUDE.md. Var bryter koden mot principerna? Var är den i linje med dem? Ge konkreta filreferenser.```

Svar
```
Jag har gått igenom src/ mot arkitekturprinciperna. Kort sagt följer koden principerna väl. Jag hittade fyra konkreta avvikelser och några gränsfall.

Avvikelser

1. Dexie läcker in i UI-lagret
useLiveQuery.ts:1 importerar liveQuery från dexie. Principen säger att data/ är det enda lagret som pratar med IndexedDB. Nu är UI:ts sätt att reagera på ändringar i databasen bundet till Dexie, och useEntries.ts, useSettings.ts och useFoodSearch.ts:34 bygger på det. Eslint fångar inte detta, eftersom eslint.config.js:27 bara begränsar importer i domain/.
Förslag: lägg en funktion watch(query, onChange) i data/ som hooken använder.

2. En komponent känner till data-lagrets feltyp
ScanScreen.tsx:2 importerar klassen OffError och gör instanceof på rad 15. Det är den enda skärmen som importerar ett värde direkt från data/ i stället för via en hook.
Förslag: låt useBarcodeLookup returnera ett eget resultat, till exempel 'offline' | 'unreachable'.

3. Komponenter räknar själva

DaySummary.tsx:6 räknar kcal kvar som goals.kcal - total.kcal. Det är en näringsberäkning i en komponent.
SplitFields.tsx:18 summerar makroandelarna själv, fast domain/goals.ts redan kontrollerar summan i isValidSplit.

4. Avrundning från UI:t hamnar i sparad data
useAmountForm.ts:29 avrundar portioner till två decimaler när man byter enhet, och sedan räknas gram fram från det avrundade värdet. Exempel: med 30 g per portion blir 10 g först 0,33 portioner och sedan 9,9 g, och det är 9,9 g som sparas. Avrundningen ligger alltså i UI:t, men den påverkar ett lagrat värde, vilket går emot tanken att avrundning bara gäller visning.

Gränsfall
Skärmar anropar domain direkt. DayScreen.tsx:15 och WeekScreen.tsx:19 anropar summarizeDay/summarizeWeek, och MealSection.tsx:30, MacroProgress.tsx:7 och SplitFields.tsx:26 anropar domain-funktioner under renderingen. De har inga egna formler, så det följer principen ordagrant. Om du menar att komponenter bara ska visa färdiga värden borde det i stället ligga i hooks som useDaySummary.
Målet sparas avrundat. useGoalsForm.ts:64 sparar det beräknade målet som Math.round(estimate). Det är rimligt för ett mål, men det är också avrundning som hamnar i sparad data.
Typer från data/ används i komponenter. FoodForm.tsx:2, FoodFormScreen.tsx:1 och BackupSection.tsx:2 importerar bara typer, vilket är ofarligt. CustomFoodInput och ImportMode skulle ändå passa bättre i domain/types.ts.
Testhjälp i produktionskoden. testData.ts ligger i domain/ men används bara i tester.
I linje med principerna
Rent domain/: inga importer av React, Dexie eller I/O, och eslint upprätthåller det (eslint.config.js:27-33). Datum skickas in i stället för att läsas inuti, till exempel createExport(..., exportedAt) i exportFormat.ts:119.
Tester: varje fil i domain/ med logik har en testfil. Bara types.ts och testData.ts saknar, och de innehåller ingen logik.
Open Food Facts mappas direkt i data/: OffProduct är privat i openFoodFacts.ts:20 och mapOffProduct returnerar Food. Livsmedelsverkets rader mappas likadant i livsmedelsverket.ts:16.
Värden per 100 g: Food och LogEntry lagrar per100g, och loggposter sparar en kopia så att historiken inte ändras (types.ts:32-45). domain/ avrundar aldrig, och visningen avrundas i format.ts:11.
Inga globala variabler: inget föränderligt state ligger på modulnivå. Databasen hålls i context (DbProvider.tsx:12), och så gör dagens datum (TodayProvider.tsx).
Komponenternas storlek: den största .tsx-filen är ScanScreen.tsx med 104 rader och den största .ts-filen router.ts med 132. Formulärlogiken är utbruten i hooks som useFoodForm, useGoalsForm och useAmountForm.
Data hämtas via hooks: alla läsningar och skrivningar mot data/ går via hooks i ui/hooks/.
Offline och nätverk: det enda anropet med fetch finns i openFoodFacts.ts:105. Produkter som hämtats från Open Food Facts sparas i IndexedDB (foodCatalog.ts:44). Livsmedelsverkets data, ungefär 135 kB, förcachas av service workern (vite.config.ts:35).
Schemamigreringar: db.ts:12-16 beskriver regeln om migreringar, och exportformatet har ett eget system för migrering i exportFormat.ts:130.

Det viktigaste att åtgärda är punkt 1, eftersom den är den enda som binder UI:t till lagringslösningen. En eslint-regel för src/ui/** som förbjuder dexie och värdeimporter från data/ utanför hooks skulle också förhindra att det händer igen.
```

## Promt 2
```Gå igenom kodbasen och leta efter: hårdkodade hemligheter eller uppgifter, saknad inputvalidering, injection-risker (SQL, kommando, path traversal), känsliga data i loggar, och osäkra standardvärden. Lista varje fynd med filnamn och radnummer.```

Svar
```
Kodbasen är i gott skick. Allt körs i webbläsaren och det enda nätverksanropet går till Open Food Facts, så det mesta i listan blir antingen inga fynd eller små fynd. Jag hittade tre riktiga brister. Ingen av dem läcker data, men de kan göra att felaktig data sparas eller att appen kraschar.

Fynd

1. Importen kontrollerar mindre än formulären gör (medel)
exportFormat.ts kontrollerar bara typerna i en importerad fil, inte samma gränser som appens formulär använder:

exportFormat.ts:66: näringsvärden har ingen övre gräns. kcal: 1e300 eller 500 g protein per 100 g går igenom, eftersom nutrientIssues från foodValidation.ts:12 inte körs.
exportFormat.ts:81-82: grams > 0 kontrolleras, men inte MAX_ENTRY_GRAMS som isValidAmount använder.
exportFormat.ts:91-92: målets kalorier kontrolleras med > 0 i stället för isValidGoalKcal (goals.ts:20).
exportFormat.ts:60-69: strängarna har ingen längdgräns. id får vara tomt för loggposter (rad 76), barcode kontrolleras inte med isValidBarcode, och updatedAt/createdAt kontrolleras inte som datum.
backupFile.ts:26: hela filen läses in och tolkas utan storleksgräns. En jättefil kan få fliken att frysa.

En fil som någon har ändrat eller som är trasig kan alltså ge orimliga summor och grafer. Det är användaren själv som väljer filen, så risken är låg, men det bryter mot principen att data ska vara giltig.

2. Data från Open Food Facts har inga övre gränser (låg–medel)
openFoodFacts.ts:45-48 godtar alla tal som är 0 eller större, och rad 71-95 sparar produkten utan att köra nutrientIssues. Produkterna i Open Food Facts är redigerade av användare, så felaktiga värden förekommer, och de cachas sedan i IndexedDB. name, brand och serving_size har heller ingen längdgräns. Det finns ingen risk för XSS, eftersom React escapar all text.

3. Felaktig %-kodning i adressen kraschar appen (låg)
router.ts:52: decodeURIComponent kastar URIError för t.ex. #/lagg-till/2026-01-01/frukost/%E0. Felet fångas av ErrorBoundary, men knappen “Ladda om” behåller hashen, så användaren fastnar tills adressen ändras för hand. Felmeddelandet (ErrorBoundary.tsx:22) säger dessutom fel sak: att datan inte kunde läsas. Ett barcode från hashen (rad 70) kontrolleras inte heller, men det fylls bara i i formuläret, som kontrollerar det innan något sparas.

Inga fynd
Hårdkodade hemligheter: inga API-nycklar, tokens eller lösenord, varken i koden eller i konfigurationen. id-token: write i deploy.yml:65 är vanlig OIDC för GitHub Pages. npm audit hittade 0 sårbarheter.
Injection:
SQL: ingen SQL. Dexie används med where().equals().
Kommandon: inget exec eller child_process.
Path traversal: ingen filåtkomst i appen. Streckkoden som byggs in i Open Food Facts-adressen kontrolleras först mot ^\d{6,14}$ (openFoodFacts.ts:119-120). Skriptet build-livsmedel.ts:35 bygger in nummer från Livsmedelsverkets API i sökvägen utan kontroll, men det körs bara för hand och mot en betrodd källa.
XSS: inget dangerouslySetInnerHTML, innerHTML eller eval. Alla href byggs med toHash från kända värden, och externa länkar har rel="noreferrer".
Känsliga data i loggar: bara console.error(e) i useAction.ts:21 och ErrorBoundary.tsx:14, och de stannar i webbläsarens konsol. Felen innehåller ingen profil- eller hälsodata.
Osäkra standardvärden: inga allvarliga. Två saker är värda att överväga:
index.html har ingen Content-Security-Policy. En <meta http-equiv="Content-Security-Policy"> som bara tillåter connect-src till world.openfoodfacts.org skulle tekniskt garantera att ingen data lämnar enheten, som CLAUDE.md lovar. GitHub Pages kan inte sätta headers, så det måste vara en meta-tagg.
.gitignore täcker *.local men inte .env. Det finns ingen .env i dag, men raden är ett billigt skydd.
```

## Promt 3
```Identifiera: funktioner som gör mer än en sak, otydlig namngivning (variabler, funktioner, klasser), saknad felhantering för externa anrop, och kod som är svår att läsa utan kommentarer. Lista de fem allvarligaste problemen.```

Svar
```
Kodbasen är överlag i gott skick. Funktionerna är små, domain/ är rent och de flesta externa anrop har felhantering. De fem allvarligaste problemen jag hittade, i ordning efter allvar:

1. Kameran kan fortsätta vara på efter att man lämnat skanningen

scanner.ts:33-35: scanNative öppnar kameraströmmen och anropar sedan video.play(). Om play() misslyckas, till exempel med AbortError när video-elementet tas bort medan kameran startar, så stängs strömmen aldrig av. Kroken i useBarcodeScanner.ts:19 visar då bara ett felmeddelande, och den får aldrig någon funktion att stänga kameran med.

Så händer det: man öppnar skanningen och skriver in en streckkod för hand innan kameran hunnit starta. Fasen byter till looking, <video> tas bort och play() misslyckas. Kameralampan lyser kvar tills sidan laddas om. Det är ett integritetsproblem och drar batteri på mobilen, som är den viktigaste plattformen.
Lösning: lägg srcObject/play() i en try som stänger av spåren om något kastas.
2. Streckkodsuppslaget kan inte avbrytas och flyttar användaren efter att hen lämnat vyn

lookupBarcode och getByBarcode tar emot en AbortSignal, men useBarcodeLookup.ts:7 skickar aldrig vidare någon. Därför anropar ScanScreen.tsx:35-36 navigate() när svaret kommer, även om användaren redan har gått därifrån.

Så händer det: man skannar på ett långsamt nät och trycker på “Tillbaka” medan det står “Söker efter …”. Upp till 10 sekunder senare hoppar appen ändå till lägg till-vyn med mängdpanelen öppen.
Lösning: skapa en AbortController per uppslag och avbryt den när komponenten avmonteras.
3. Exporten säger “Filen är sparad.” utan att veta om den blev det

backupFile.ts:16-18 återkallar blob-URL:en direkt efter link.click(). BackupSection.tsx:31-34 visar sedan alltid ett lyckat resultat. Eftersom datan bara finns på enheten är exporten användarens enda säkerhetskopia, så ett felaktigt “klart” kostar mycket här. Blob-nedladdningar har länge varit opålitliga i iOS Safari och särskilt i installerade PWA:er.

Overifierat: jag har inte testat detta på en iPhone, så det behöver provas på enheten.
Lösning: vänta med revokeObjectURL (till exempel med setTimeout), använd gärna navigator.share med en fil på iOS, och formulera meddelandet som “Filen har skapats” i stället för att lova att den sparats.
4. Fel från externa anrop sorteras in i fel kategori
openFoodFacts.ts:114: response.json() ligger utanför try. Ogiltig JSON eller en timeout medan svaret läses kastar då SyntaxError eller DOMException i stället för OffError. Det bryter mot det som står i dokumentationen (“Kastar OffError om Open Food Facts inte går att nå”).
foodCatalog.ts:44: om cacheOffFoods misslyckas, till exempel för att lagringsutrymmet är fullt, visar ScanScreen.tsx:14-17 “Open Food Facts gick inte att nå just nu”, trots att hämtningen lyckades och felet ligger lokalt.
Lösning: flytta .json() in i felmappningen, och skilj på fel från Open Food Facts och fel vid lokal lagring i errorText.
5. Otydliga namn där samma ord betyder olika saker
FoodFormScreen.tsx:22, 43: variabeln source är livsmedlet som kopieras eller ändras, så source.source !== 'custom' blandar ihop “ursprungslivsmedel” och “datakälla”. baseFood eller original vore tydligare.
AddFoodScreen.tsx:23 entries och FoodFormScreen.tsx:23 foods innehåller skrivfunktioner, inte poster eller livsmedel. entries.add(...) läses som en array. entryActions och foodActions vore tydligare.
router.ts:57-58: first…fourth betyder olika saker i olika rutter. first är ett datum för vecka men en action-slug för livsmedel, och ändå tolkas date från first för alla rutter.

Mindre saker:

useGoalsForm.ts:64 gör tolkning, validering, uppskattning och lägeslogik på en tät rad som är svår att följa utan en kommentar.
useSaveGoals.ts:9-10 sparar mål och profil utan transaktion, så ett fel mitt emellan kan spara bara det ena.
```

## Åtgärd

Jag bad agenten åtgärda de fel som den själv föreslig var viktigast.

Ändringarna finns commitade i projektet

* [08453ae](https://github.com/tobiaso88/mumsa/commit/08453ae) Stop camera on failed start and cancel lookups when leaving scan
* [8aaeede](https://github.com/tobiaso88/mumsa/commit/8aaeede) Validate imports, Open Food Facts data and routes against app limits
* [1c2b23d](https://github.com/tobiaso88/mumsa/commit/1c2b23d) Enforce layer boundaries between ui and data
