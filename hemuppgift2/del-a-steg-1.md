# Hemuppgift 2, Del A - Steg 1

# Kodanalys: läsbarhet, duplicering och teknisk skuld

Analys av `src/domain/`, `src/data/` och `src/ui/` per 2026-10-27. Ingen kod har ändrats.

Kodbasen är överlag i gott skick. Lagren hålls isär, `domain/` är rent och har tester, och kommentarerna förklarar varför koden gör som den gör. Fynden nedan är ordnade efter hur mycket skuld de kan ge längre fram.

## Störst risk

### 1. Fel visas aldrig för användaren

- `src/ui/hooks/useLiveQuery.ts:18` skriver bara fel till konsolen. Resultatet stannar då på `undefined`, och skärmar som `DayScreen`, `WeekScreen` och `ProfileScreen` visar `return null`. Användaren ser en tom skärm utan förklaring.
- Ingen av de asynkrona händelsehanterarna har try/catch. Det gäller `GoalsForm.handleSubmit`, `EditEntrySheet`, `AddFoodScreen.handleAdd`, `FoodFormScreen.handleSubmit` och `handleDelete`, `BackupSection.handleExport` och `runImport`. Om IndexedDB fallerar, till exempel när lagringen är full i Safaris privata läge, händer ingenting synligt.
- Det finns ingen gemensam laddningsvy. "Laddar" och "fel" ser likadana ut.

### 2. Affärsregler ligger i UI-lagret

CLAUDE.md säger att UI:t inte ska räkna själv, men flera regler ligger i hooks som saknar enhetstester:

- Kalorimålet 500–10 000 kcal ligger i `src/ui/hooks/useGoalsForm.ts:65`.
- Maxgränsen 10 000 g ligger i `src/ui/hooks/useAmountForm.ts:21`.
- Omräkningen från gram till portioner, `grams / serving`, finns i `src/ui/hooks/useAmountForm.ts:29`. Det är motsatsen till `portionsToGrams` i domain, men den ligger här.
- Felmeddelandena i `src/ui/components/FoodForm.tsx:7-11` säger "100 g" och "cirka 900 kcal". Gränserna i `src/domain/foodValidation.ts:11-12` är 105 och 950. Texterna och gränserna hänger inte ihop och kommer att glida isär ännu mer.

### 3. Streckkodsregeln och id-prefixen är kopierade

- Regexen `/^\d{6,14}$/` finns på tre ställen: `src/data/openFoodFacts.ts:117`, `src/ui/hooks/useFoodForm.ts:54` och `src/ui/screens/ScanScreen.tsx:30`. Blankstegen rensas också bort på två ställen. Det behövs en `normalizeBarcode` och en `isValidBarcode` i `domain/`.
- Prefixen `'slv:'`, `custom:` och `off:` är hårdkodade strängar. `src/data/foodCatalog.ts:19-27` använder `startsWith('slv:')` tre gånger för att avgöra var ett livsmedel finns, trots att `source` redan säger det. Om formatet ändras går uppslagen sönder utan att något larmar.

### 4. Valideringen av importer följer inte typerna

- `src/domain/exportFormat.ts:46` listar `'off' | 'slv' | 'custom'` för hand, och rad 88 gör samma sak med kön. Om en ny `FoodSource` läggs till godtas den inte av valideringen, och kompilatorn säger ingenting. Det går att lösa som för `MEALS` och `ACTIVITY_LEVELS`, med `FOOD_SOURCES` och `SEXES` som `as const`-arrayer. Samma lista finns också i `src/ui/components/ProfileFields.tsx:32`.
- Importen godtar livsmedel med `source: 'slv'` och sparar dem i `db.foods`. Sedan visas de dubbelt i sökningen, eftersom `searchAllFoods` (`src/data/foodCatalog.ts:13`) slår ihop sparade livsmedel med den buntade databasen. Den kontrollerar heller inte att id är unika eller att prefix och `source` stämmer överens.
- Versionen kontrolleras med `version !== EXPORT_VERSION` och typen `typeof EXPORT_VERSION`. Det finns ingen plats att migrera gamla exporter. Första gången formatet ändras slutar alla befintliga säkerhetskopior att fungera, om inte en migreringskedja byggs in först.

### 5. Prestanda vid varje tangenttryck

Varje sökning läser in `db.foods.toArray()`, mappar om cirka 2 500 rader från Livsmedelsverket till nya objekt i `src/data/livsmedelsverket.ts:32` och kör `toLocaleLowerCase` två gånger per livsmedel. Det tål dagens mängd, men det blir långsammare för varje skannad produkt som sparas. `findFood` söker också linjärt genom alla rader. En memoiserad lista med normaliserad söktext och en `Map` på id räcker.

## Duplicerad logik

### 6. Samma asynkrona mönster finns sex gånger

`useEffect` + `let cancelled` + `.then` + `setState` finns i:

- `src/ui/screens/AddFoodScreen.tsx:25`
- `src/ui/screens/FoodFormScreen.tsx:22`
- `src/ui/hooks/useFoodSearch.ts:15` och rad 39
- `src/ui/hooks/usePersistentStorage.ts:8`
- `src/ui/hooks/useBarcodeScanner.ts:9`

En hook som `useAsync(fn, deps)` skulle ta bort mönstret och ge felhanteringen från punkt 1 ett naturligt ställe att bo på.

### 7. Hur UI:t når datalagret skiljer sig åt

Läsningar går oftast via hooks, men `AddFoodScreen` och `FoodFormScreen` anropar `findFood` direkt. Skrivningar görs direkt från komponenter som `GoalsForm`, `EditEntrySheet` och `BackupSection`. Det strider mot principen att komponenter går via hooks, och det gör punkt 1 svårare att lösa på ett ställe.

### 8. Upprepade bitar i UI:t

- Fyra `ProgressBar` och `macroGrams` är kopierade mellan `src/ui/components/DaySummary.tsx:18-21` och `src/ui/screens/WeekScreen.tsx:48-51`. De kan bli en `MacroProgress`-komponent.
- Rubriken med föregående och nästa är kopierad mellan `src/ui/components/DayHeader.tsx` och `src/ui/screens/WeekScreen.tsx:22-35`. Båda använder klassen `day-header`.
- Segmenterade radioknappar är handskrivna fyra gånger, i `AmountForm`, `BackupSection`, `GoalsForm` och `ProfileFields`. De har hårdkodade `name`-attribut (`"unit"`, `"mode"`, `"sex"`), som krockar om två formulär visas samtidigt.
- `TextField` inuti `src/ui/components/FoodForm.tsx:13` är nästan identisk med `NumberField`.
- Importfilen valideras två gånger: först i `src/ui/hooks/useImport.ts:22`, sedan igen i `importAll`. Statet sparar både rå `json` och validerad `data`. `ImportOutcome` och `ImportResult` har dessutom nästan samma form.
- `src/data/backup.ts:40-41` skriver till `settings` direkt med egna nyckelsträngar i stället för att använda `saveGoals` och `saveProfile`.

## Läsbarhet

- **`ref` som prop-namn.** `FoodFormScreen` tar emot `ref`, och `src/ui/App.tsx:32` sprider `{...route}` in i komponenten. Det krockar med Reacts reserverade `ref`. Det fungerar i React 19, där `ref` är en vanlig prop, men det är förvirrande och skört. Fältet betyder dessutom olika saker beroende på `action`: en streckkod för `new` och ett livsmedels-id för `copy` och `edit`. Två separata fält, till exempel `barcode` och `foodId`, vore tydligare.
- **Nästlade ternarier** i `src/ui/screens/AddFoodScreen.tsx:63-78`, där `searching ? results && (...) : recent && ...` är svår att följa. Samma sak i `src/ui/components/SplitFields.tsx:26`.
- **Långa rader** med flera tillståndsövergångar, till exempel `src/ui/hooks/useImport.ts:23` och `:29`. Feltexten "inte giltig JSON" avgörs med `json === null`, så en fil som innehåller exakt `null` får fel meddelande.
- **Olika sätt att göra samma sak:** `useId` i vissa komponenter men hårdkodade id som `import-file` och `manual-barcode` i andra. Procentfälten använder `String()` medan övriga fält använder `toInputText`, så decimalpunkt och decimalkomma blandas.
- **Datum i renderingen:** `new Date()` anropas på flera ställen, bland annat `useRoute`, `BottomNav`, `DayHeader` och `WeekScreen`. "Idag" byts inte vid midnatt om appen ligger öppen, och det gör tester beroende av klockan.
- **`components.css` är 615 rader i en enda fil.** Den följer inte 150-radersprincipen som gäller för komponenter.

## Testtäckning

`useGoalsForm`, `useAmountForm` och `useFoodForm` har inga egna tester, trots att de innehåller valideringen från punkt 2. De testas bara indirekt via skärmtesterna. Om reglerna flyttas till `domain/` blir de täckta av kravet att allt i `domain/` ska ha enhetstester.

## Förslag på ordning

1. Punkt 1–3. De påverkar användaren direkt eller sprider sig mest om de får vara kvar.
2. Punkt 4, innan export- eller databasformatet ändras första gången.
3. Resten är städning som kan göras efter hand.

