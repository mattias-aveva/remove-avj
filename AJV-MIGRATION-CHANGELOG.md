# Changelog: AJV-migrering

## Sammanfattning

Frontend-valideringen har flyttats från AJV till en egen TypeScript-validator för att uppfylla CSP-kravet `script-src 'self'` utan `unsafe-eval`.

Frontend-validering och omedelbar feedback till användaren finns kvar.

## Genomförda ändringar

### Ny intern validator

Skapad:

- `src/features/auto-ui/schema-validator.ts`

Validatorn använder ingen `eval`, `new Function` eller extern valideringsbibliotek. Den stöder de JSON Schema-regler som används av projektet, bland annat:

- typer och `null`
- `required`, `properties`, `items`
- `allOf`, `anyOf`, `oneOf`, `not`
- `$ref`, `enum`, `const`
- `pattern`, textlängd och numeriska gränser
- `if/then`, `dependentRequired`
- `guid`, `uuid` och `int32`
- listor, objekt och `additionalProperties`

### Migrerade valideringsflöden

Följande använder nu den interna validatorn:

- Auto UI-inställningar i `auto-ui-validation.ts`
- Flow Module-inställningar i `module-setting-validation.ts`
- Resource Draft-data i `validation-resource-version-data.ts`

Befintlig funktionalitet har behållits, inklusive:

- `null`-hantering
- Universal Connector-parametrar
- `commonSettings`
- resource-kontroller
- GUID-feltexten `select a valid item from the list`
- användarens befintliga felstruktur

### Felhantering

- Nästlade `required`-fel behåller full sökväg, exempelvis `simpleSettings/requiredStringSetting`.
- Nästlade objekt-, dictionary- och arrayfel matchas korrekt i Auto UI.
- Sibling-fält med liknande namn matchas inte felaktigt.
- Olösta `$ref` ger ett explicit valideringsfel.
- `int32` valideras inom intervallet `-2147483648` till `2147483647`.
- Resource-validering hanterar även fall där endast ett `not`-fel returneras.

## Tester

Nya och uppdaterade tester täcker:

- Auto UI-validering mot backend-schema
- Flow Module- och Universal Connector-validering
- Resource Draft för Modbus, DataTrigger och Aveva Historian
- `$ref`, `if/then`, `dependentRequired` och `not`
- nästlade required-, objekt-, dictionary- och arrayfel
- `int32`-gränser
- UI-matchning av valideringsfel
- JSON-parsning och ogiltiga inputtyper

## Dependencies och cleanup

- AJV-importer och AJV-anrop är borttagna från produktionskoden.
- AJV-specifika mockar och tester är borttagna.
- `ajv` och `ajv-draft-04` är borttagna från `package.json` och `package-lock.json` som direkta dependencies.
- Eventuella transitiva AJV-versioner lämnas kvar när andra paket kräver dem.

## Verifiering

Genomförda kontroller:

- Automatiserade tester: `950` passerade, `1` skip
- ESLint: passerar
- TypeScript: passerar
- Produktionsbuild: passerar
- Användartester: genomförda med gott resultat
- `package-lock.json`: giltig och utan ändrade paketversioner

## Status

AJV är funktionellt ersatt i projektets tre valideringsflöden. Ändringen är testad automatiskt och manuellt och är redo för pull request.
