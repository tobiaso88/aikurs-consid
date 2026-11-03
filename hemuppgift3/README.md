# Hemuppgift 3, Del A

## Problemet jag analyserade
https://github.com/laravel/framework/issues/54401

## Hur jag beskrev det för agenten
Jag tog ut issue'n som en markdown fil och bifogade den tillsammans med promten: `Analysera det här felet och formulera tre hypoteser om rotorsaken, ordnade efter sannolikhet. För varje hypotes: förklara mekanismen och ange konkreta verifieringssteg.`

## Agentens hypoteser

- Hypotes 1: Eager loading går via whereIntegerInRaw (mest sannolik)
- Hypotes 2: PDO binder heltalet som PARAM_INT och pgsql typar det som integer
- Hypotes 3: Schemat är inkonsekvent, och issuet är egentligen ett feature request

Sammanfattning: Hypotes 1 förklarar exakt den observerade SQL:en (in (1)). De andra två kräver ovanliga förutsättningar eller förklarar varför felet bara syns på pgsql. Ett snabbt första steg är att be rapportören om stacktrace och det faktiska anropet. Det avgör direkt om felet kommer från addEagerConstraints.

## Vilken hypotes jag valde och varför
Jag valde hypotes 3 då det inte finns något sett för ramverket att veta om id i detta fallet är en sträng då det inte defineras någonstans.

## Vad som faktiskt var fel
Det är inte en bugg i ramverket utan en designbegränsning.
I migrationen definierades owner_id manuellt som en sträng i stället för med morphs(), som hade gett samma typ som users.id.
Eloquent förutsätter att den polymorfa kolumnen och förälderns primärnyckel har samma typ, och binder därför id:t som ett heltal.
MySQL konverterar typen tyst, men PostgreSQL kräver att typerna matchar och ger ett fel.
Laravels maintainer har bedömt att lösningen är att låta typerna matcha, inte att ändra ramverket.

## Vad agenten fick rätt och vad den missade

Vad agenten fick rätt

- Hittade den tekniska mekanismen: Eloquent väljer whereIntegerInRaw utifrån förälderns nyckeltyp och bryr sig inte om kolumnens typ.
- Såg att in (1) i felet pekar på eager loading, och inte på stegen i reproduktionen.
- Hypotes 3 (felaktigt schema, alltså en designbegränsning och ingen bugg) stämde med maintainerns bedömning.

Vad agenten missade

- Kollade inte GitHub först, trots att issuet och flera stängda PR:er redan fanns där.
- Rankade hypotes 3 lägst. Den beskrev egentligen varför felet inte lagas, medan hypotes 1 beskrev hur det uppstår, så båda stämde.
- Avfärdade lazy loading för lätt. Alla fix-PR:er castar även den vägen, men det är inte verifierat.
- Föreslog en lösning ($keyType = 'string') som påverkar alla relationer, trots att rapportören behövde en lösning för bara en relation.
