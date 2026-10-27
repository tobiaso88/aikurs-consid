# Del A - Generell implementation med plan och implementation

1. Vad identifierade agenten?

Kodbasen är i gott skick överlag, men den hittade problem som kan bli teknisk skuld:

* Fel visas aldrig för användaren. Läsfel ger en tom skärm, och skrivfel syns inte alls.
* Affärsregler ligger i UI:t, till exempel gränser för kalorier och gram. Felmeddelandena stämmer inte med gränserna i domain/.
* Streckkodsregeln finns på tre ställen, och id-prefixen som slv: är hårdkodade strängar.
* Importvalideringen följer inte typerna, kan ge dubbletter i sökningen och har ingen väg för att migrera gamla exporter.
* Sökningen räknar om cirka 2 500 livsmedel vid varje tangenttryck.
* Samma asynkrona mönster med cancelled är kopierat sex gånger.
* Komponenter anropar datalagret direkt, ibland via hooks och ibland inte.
* Flera dubbletter i UI:t, bland annat makrostaplarna, rubriken med föregående och nästa, de segmenterade radioknapparna och textfälten.

2. Vad ville agenten göra som du stoppade och varför?

Agenten tog fram en plan om 6 faser för att fixa det som var bristfälligt.
Jag behövde inte stoppa den från att göra något, utan den väntade på att jag skulle be den köra igång.
Jag tycker att planen kändes rimlig.

3. Vad missade den

Jag har aldrig byggt något i ren react tidigare, så jag har inte mycket erfarenhet om vad den kan ha missat.

4. Är projektet körbart efter implementationen?

Jag har testat applikationen efter varje fas, och den fungerar.

## Avvikelser

### Fas 1

Avvikelser från planen

* useExport är en ny hook som inte fanns med i planen. Den behövdes för att exporten inte skulle anropa datalagret direkt.
* useBarcodeScanner är oförändrad. Den hanterar redan sina fel och passade inte mönstret i useAsync, vilket planen också räknade med.
* Om isStoragePersisted() misslyckas tolkas det som att webbläsaren saknar stöd. Annars skulle ett fel i en sidofunktion ersätta hela appen med felvyn.

### Fas 3

När agenten skulle påbörja fas 3 så upptäckte den att det inte skulle ge speciellt mycket prestandavinst att genomföra ändringen.
Den föreslog då att vi skulle skjuta upp denna uppgift.

### Fas 4

Här hittade agenten att den missat saker i fas 1:

Missat i fas 1: ScanScreen anropar fortfarande lookupBarcode direkt i stället för via en hook. Den låg inte med bland filerna i fas 1, så jag har inte ändrat den här. Den kan tas med i fas 5.

### Fas 5

Uppdelningen av components.css är struken, eftersom jag missade att svara på beslut 4 i planen. Agenten hade föreslagit att stryka den.

Planen beskrev useToday som en fristående hook. Nu ligger datumet i context, så att alla vyer delar samma värde.
