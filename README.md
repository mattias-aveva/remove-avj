# Auto UI Validation

[AJV migration change log](AJV-MIGRATION-CHANGELOG.md)

## Syfte

Auto UI är ett schema-drivet formulärsystem. JSON Scheman beskriver modulernas inställningar och används för att:

- skapa formulärfält automatiskt
- sätta standardvärden
- hantera objekt, listor, dictionaries, credentials och resources
- validera användarens input i frontend
- visa fel vid rätt inställning

Frontend-valideringen behålls eftersom användaren behöver omedelbar återkoppling under konfiguration.

## CSP-bakgrund

Hostingmiljön använder:

```text
script-src 'self'
```

och tillåter inte `unsafe-eval`.

AJV ersattes eftersom runtime-kompilering av JSON Schema kan använda dynamiskt genererad kod. Den nya lösningen använder vanlig TypeScript-logik och ingen `eval`, `new Function` eller runtime-kodgenerering.

## Arkitektur

```text
JSON Schema
    |
    v
PropertyInfo och formulärstruktur
    |
    v
Auto UI-komponenter
    |
    v
Användarens settings
    |
    v
Intern TypeScript-validator
    |
    v
Valideringsfel tillbaka till UI
```

### Schemahantering

- `src/features/auto-ui/auto-ui-schema.ts`
- `src/features/auto-ui/module-schema.ts`
- `src/features/auto-ui/types-auto-ui.ts`

Dessa filer tolkar JSON Schema och skapar `PropertyInfo`, som innehåller typ, standardvärde, enum, struktur och UI-metadata.

### Rendering

- `src/modules/AutoUi/AutoUi.vue`
- `src/modules/AutoUi/components/`

`AutoUi.vue` renderar rätt inputkomponent för varje `PropertyInfo`. Ändringar skickas upp som `settingUpdate`. Validering begärs efter ändring med cirka 300 ms fördröjning.

## Validering

### Gemensam validator

`src/features/auto-ui/schema-validator.ts`

Den generella validatorn returnerar interna fel:

```ts
{
  instancePath: string
  keyword: string
  params: Record<string, unknown>
  message: string
}
```

Den stöder projektets relevanta JSON Schema-funktionalitet:

- typer och nullable `null`
- `required`, `properties`, `items`
- `additionalProperties`
- `allOf`, `anyOf`, `oneOf`, `not`
- `enum` och `const`
- `pattern`, `minLength`, `maxLength`
- `minimum`, `maximum`
- `exclusiveMinimum`, `exclusiveMaximum`
- `multipleOf`, `minItems`, `maxItems`, `uniqueItems`
- lokala `$ref`
- `if/then`
- `dependentRequired`
- `guid` och `uuid`
- `int32`

`int32` accepterar intervallet:

```text
-2147483648 <= value <= 2147483647
```

Olösta lokala `$ref` returnerar ett explicit fel. De godkänns inte tyst.

### Auto UI-adapter

`src/features/auto-ui/auto-ui-validation.ts`

`validateAutoUiSettings(...)` använder den gemensamma validatorn och anpassar felen till:

```ts
{
  setting: string
  errorMessage: string
}
```

Adaptern behåller:

- `null` till `undefined` för icke-nullable inställningar
- fullständiga sökvägar för nested `required`-fel
- GUID-meddelandet `select a valid item from the list`
- hantering av flera fel
- loggning och skydd mot interna valideringsfel

### Flow Module-validering

`src/features/auto-ui/module-setting-validation.ts`

Används för modulinställningar och Universal Connector-parametrar. Den behåller logik för:

- sammanslagning av `allOf`
- SDK-baserade `commonSettings`
- dynamiska Universal Connector-scheman
- resource-versioner som är arkiverade eller saknar data
- validering av enskilda nästlade element

### Resource Draft

`src/features/resource-version/validation-resource-version-data.ts`

Används för resource-typerna:

- `Modbus`
- `DataTriggerDefinitions`
- `AvevaHistorian`

Funktionen behåller befintliga feltexter för ogiltig JSON, felaktig inputtyp och schemafel:

```text
Failed to parse data. Expected JSON.
Unexpected format of resource data.
Resource contains error: ...
```

## Felvägar till UI

Valideringsfel använder slash-separerade paths:

```text
simpleSettings/requiredStringSetting
listSettings/listOfObjects/0/value
dictionary/key/nestedField
```

`src/modules/AutoUi/utils/validation-error.ts` matchar nu både exakta paths och alla verkliga child paths under en setting. Matchningen är boundary-aware, så exempelvis `settings/key` inte matchar `settings/keyOther`.

## Tester

### Viktiga testfiler

- `src/features/auto-ui/schema-validator.test.ts`
- `src/features/auto-ui/test/auto-ui-validation.test.ts`
- `src/features/auto-ui/test/auto-ui-validation.integration.test.ts`
- `src/features/auto-ui/test/module-validation.test.ts`
- `src/features/resource-version/validation-resource-version-data.test.ts`
- `src/modules/AutoUi/utils/validation-error.test.ts`

Test-fixturen `validation-test-module.schema.json` kommer från backend-formatet och används för att testa realistiska schemas.

Testerna täcker bland annat:

- giltiga och ogiltiga värden
- required och nullable
- nested paths
- enum, pattern och längdgränser
- numeriska gränser och `int32`
- GUID/UUID
- objekt, dictionaries och arrayer
- `$ref`, `oneOf`, `anyOf`, `not`, `if/then` och `dependentRequired`
- Resource Draft-parsning och schemafel
- UI-matchning av nästlade fel

## Verifiering

Senaste verifierade status:

```text
Hela testsuiten: 950 tester passerade, 1 skip
ESLint: passerar
TypeScript: passerar
Produktionsbuild: passerar
Användartester: genomförda med gott resultat
```

Testresultat kan ändras när nya tester läggs till. Aktuell terminaloutput är alltid den slutliga källan för exakta totalsiffror.

## Dependencies

AJV och `ajv-draft-04` är borttagna som direkta dependencies från:

- `package.json`
- `package-lock.json`

Transitiva AJV-versioner kan fortfarande finnas i lockfilen om andra verktyg kräver dem. De ska inte tas bort manuellt, eftersom det kan bryta exempelvis build-, lint- eller editorpaket.

## Underhåll

Använd följande filer för framtida ändringar:

| Ändring | Fil |
|---|---|
| Generell JSON Schema-regel | `src/features/auto-ui/schema-validator.ts` |
| Auto UI-fel och null-hantering | `src/features/auto-ui/auto-ui-validation.ts` |
| Modul- och Universal Connector-logik | `src/features/auto-ui/module-setting-validation.ts` |
| Resource Draft | `src/features/resource-version/validation-resource-version-data.ts` |
| UI-matchning av fel | `src/modules/AutoUi/utils/validation-error.ts` |

Lägg alltid till ett fokuserat test när en ny schema-regel eller felväg införs.

## Rekommenderade kommandon

```bash
npm run test:output
npx tsc --noEmit
npm run lint
npm run build
```

Kör fokuserade tester med exempelvis:

```bash
npm run test:output -- src/features/auto-ui/test
npm run test:output -- src/features/resource-version/validation-resource-version-data.test.ts
npm run test:output -- src/modules/AutoUi/utils/validation-error.test.ts
```

## Slutstatus

Den funktionella AJV-migreringen är genomförd och verifierad med automatiserade tester, build och användartester. Projektets produktionsvalidering använder den interna TypeScript-validatorn och kräver inte `unsafe-eval` för denna valideringsfunktion.
