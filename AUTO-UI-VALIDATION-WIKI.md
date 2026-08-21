# Auto UI Validation

## Overview

Auto UI is a schema-driven form system. Module and resource settings are described with JSON Schema, and the frontend uses those schemas to:

- build the correct input controls
- create default settings
- handle nested objects, lists, dictionaries, credentials, and resources
- validate user input before it is submitted
- display validation errors next to the correct setting

The validation engine is implemented internally in TypeScript so validation can run in the browser without runtime code generation.

## Why AJV Was Replaced

The hosting environment enforces the following Content Security Policy:

```text
script-src 'self'
```

without `unsafe-eval`.

AJV can compile JSON Schemas at runtime using generated functions. That is incompatible with the policy. Backend-only validation is not sufficient because users need immediate feedback while configuring a module.

The selected solution is therefore:

```text
JSON Schema + settings
        |
        v
Internal TypeScript validator
        |
        v
Validation errors for Auto UI
```

The validator does not use `eval`, `new Function`, or another external validation library.

## Architecture

### Schema interpretation

The schema is first interpreted into `PropertyInfo` objects. These objects contain the information needed to build the form:

- setting type
- title and description
- default value
- enum values
- list and object structure
- resource and credential metadata
- sorting and UI metadata

Main files:

- `src/features/auto-ui/auto-ui-schema.ts`
- `src/features/auto-ui/module-schema.ts`
- `src/features/auto-ui/types-auto-ui.ts`

### Form rendering

`src/modules/AutoUi/AutoUi.vue` receives the schema-derived properties and current settings. It selects and renders dynamic controls such as:

- text inputs
- boolean inputs
- enum selectors
- resource selectors
- credential selectors
- code editors
- list controls
- dictionary controls
- nested object controls

The form emits setting updates. Validation is requested after changes, with a short throttle to avoid validating on every rapid input event.

### Validation

The validation engine and adapters are separated:

```text
schema-validator.ts
    General JSON Schema validation

    |
    +--> auto-ui-validation.ts
    |       Auto UI error mapping and null handling
    |
    +--> module-setting-validation.ts
    |       Flow module and Universal Connector behavior
    |
    +--> validation-resource-version-data.ts
            Resource Draft behavior
```

## Main Files

### `schema-validator.ts`

Path:

`src/features/auto-ui/schema-validator.ts`

This is the general validation engine. It returns internal errors in this format:

```ts
{
  instancePath: string
  keyword: string
  params: Record<string, unknown>
  message: string
}
```

Supported functionality includes:

- `type`
- nullable types using `null`
- `required`
- `properties`
- `items`
- `additionalProperties`
- `allOf`
- `anyOf`
- `oneOf`
- `not`
- `enum`
- `const`
- `minLength`
- `maxLength`
- `pattern`
- `minimum`
- `maximum`
- `exclusiveMinimum`
- `exclusiveMaximum`
- `multipleOf`
- `minItems`
- `maxItems`
- `uniqueItems`
- local `$ref`
- `if` / `then`
- `dependentRequired`
- `guid` and `uuid`
- `int32`

The `int32` format accepts the inclusive range:

```text
-2147483648 <= value <= 2147483647
```

Unresolved local references return an explicit validation error. They must not silently make a value valid.

### `auto-ui-validation.ts`

Path:

`src/features/auto-ui/auto-ui-validation.ts`

This is the Auto UI adapter. It:

1. adapts `null` values for non-nullable settings
2. calls `validateJsonSchema(...)`
3. maps internal errors to `AutoUiValidationError`
4. creates complete setting paths for required errors
5. provides user-friendly GUID messages
6. catches and logs unexpected validation failures

The public function used by Auto UI is:

```ts
validateAutoUiSettings(schema, settings)
```

The UI-facing error format is:

```ts
{
  setting: string
  errorMessage: string
}
```

Example:

```text
Schema path:   /commonSettings/foo
Missing field: bar
UI path:       commonSettings/foo/bar
```

### `module-setting-validation.ts`

Path:

`src/features/auto-ui/module-setting-validation.ts`

This file adds flow-module-specific behavior:

- merges `allOf` schemas with `json-schema-merge-allof`
- replaces `commonSettings` with the SDK schema
- creates dynamic Universal Connector parameter schemas
- validates flow-module settings
- checks archived and incomplete resource versions
- validates individual nested element properties

The nested property lookup must support schemas both before and after `allOf` merging. Merged schemas may no longer contain an `allOf` property, so lookups search through `properties`, `items`, `allOf`, `oneOf`, and `anyOf`.

### `validation-resource-version-data.ts`

Path:

`src/features/resource-version/validation-resource-version-data.ts`

This file validates Resource Draft data for the resource types listed in:

`src/features/resource-type/resource-type-types.ts`

Currently schema validation is enabled for:

- `Modbus`
- `DataTriggerDefinitions`
- `AvevaHistorian`

It preserves the existing public behavior for:

- invalid JSON
- unsupported input types
- resource types without schema validation
- user-facing resource validation messages

### `validation-error.ts`

Path:

`src/modules/AutoUi/utils/validation-error.ts`

This helper connects validation paths to UI components. A setting now matches:

- its exact path
- any nested child path below it
- array item paths
- dictionary paths

The match is path-boundary aware, so `settings/key` does not incorrectly match `settings/keyOther`.

## Validation Flow

### Flow module settings

```text
Module schema
    |
    v
Schema merge and module-specific adjustments
    |
    v
Current module settings
    |
    v
validateJsonSchema(...)
    |
    v
AutoUiValidationError[]
    |
    v
FlowModuleAutoUi -> AutoUi -> input components
```

### Resource Draft

```text
Resource Draft input
    |
    v
Parse JSON if necessary
    |
    v
Select resource schema
    |
    v
validateJsonSchema(...)
    |
    v
Resource contains error: ...
```

### Validation timing

`AutoUi.vue` emits a validation request after settings change. The request is throttled by approximately 300 ms. The parent component performs validation and sends the resulting errors back through the `validation-errors` prop.

This keeps rendering and validation separate:

- Auto UI renders fields.
- The parent validation adapter validates settings.
- The input components display matching errors.

## Error Path Rules

Validation paths use slash-separated segments:

```text
simpleSettings/requiredStringSetting
listSettings/listOfObjects/0/value
dictionary/key/nestedField
```

Required-property errors must combine the object path and missing property:

```text
instancePath:      /simpleSettings
missingProperty:   requiredStringSetting
resulting setting: simpleSettings/requiredStringSetting
```

This is required because UI components match errors using complete paths.

## Test Strategy

### Characterization tests

Before replacing AJV, representative schemas and expected results were captured. The backend fixture is:

`src/features/auto-ui/test/test-data/validation-test-module.schema.json`

The fixture covers:

- `allOf`
- `oneOf`
- nullable values
- required fields
- enums
- patterns
- string lengths
- numeric ranges
- GUID values
- nested objects
- arrays
- additional properties

### Main test files

- `src/features/auto-ui/test/auto-ui-validation.test.ts`
- `src/features/auto-ui/test/auto-ui-validation.integration.test.ts`
- `src/features/auto-ui/test/module-validation.test.ts`
- `src/features/auto-ui/schema-validator.test.ts`
- `src/features/resource-version/validation-resource-version-data.test.ts`
- `src/modules/AutoUi/utils/validation-error.test.ts`

### Important regression cases

Tests cover:

- nested required paths
- nested array and dictionary errors
- sibling-path exclusion
- `int32` boundaries
- unresolved `$ref`
- `not`-only errors
- conditional Modbus validation
- DataTrigger dependencies
- Resource Draft parsing and schema selection

### Recommended commands

Run all tests:

```bash
npm run test:output
```

Run Auto UI tests:

```bash
npm run test:output -- src/features/auto-ui/test
```

Run Resource Draft tests:

```bash
npm run test:output -- src/features/resource-version/validation-resource-version-data.test.ts
```

Run UI error matching tests:

```bash
npm run test:output -- src/modules/AutoUi/utils/validation-error.test.ts
```

Run TypeScript validation:

```bash
npx tsc --noEmit
```

Run lint for affected files:

```bash
npx eslint src/features/auto-ui/auto-ui-validation.ts src/features/auto-ui/module-setting-validation.ts src/features/auto-ui/schema-validator.ts src/features/resource-version/validation-resource-version-data.ts src/modules/AutoUi/utils/validation-error.ts --max-warnings 0
```

Build the production bundle:

```bash
npm run build
```

## Current Verification Status

The migration work has been verified with:

- full Jest test suite passing, with one existing skipped test
- Auto UI tests passing
- Resource Draft tests passing
- UI validation matching tests passing
- TypeScript passing
- ESLint passing for affected files
- production build passing

The exact totals can change as new tests are added. Always use the latest command output as the source of truth.

## Manual User Verification

Automated tests do not fully replace browser validation. The following workflow should be tested in the target environment:

1. Add the `Validation Test` module to a flow.
2. Confirm validation errors are displayed immediately for invalid initial settings.
3. Edit invalid fields and verify that the corresponding error updates or disappears.
4. Test nested object, dictionary, and array settings.
5. Test credential and resource selectors with invalid or missing values.
6. Open Resource Draft and validate Modbus, DataTrigger, and Aveva Historian data.
7. Inspect the browser console for CSP errors.
8. Inspect the production response headers and confirm there is no requirement for `unsafe-eval`.

## Remaining Cleanup

The runtime validation paths use the internal validator, but AJV cleanup should be verified separately before removing dependencies.

Search for remaining references:

```bash
rg "ajv|Ajv|unsafe-eval" src package.json package-lock.json
```

Remaining cleanup should include:

- remove unused AJV imports and helpers
- remove AJV-specific mocks and obsolete tests
- remove `ajv` and `ajv-draft-04` from `package.json`
- update `package-lock.json`
- run the full test suite again
- run lint, TypeScript, and production build
- verify CSP in the deployed browser environment

## Maintenance Guide

Use these files for future changes:

| Change needed | File |
|---|---|
| General JSON Schema rule | `src/features/auto-ui/schema-validator.ts` |
| Auto UI error mapping | `src/features/auto-ui/auto-ui-validation.ts` |
| Flow module-specific behavior | `src/features/auto-ui/module-setting-validation.ts` |
| Resource Draft behavior | `src/features/resource-version/validation-resource-version-data.ts` |
| UI error-to-field matching | `src/modules/AutoUi/utils/validation-error.ts` |
| Schema validator tests | `src/features/auto-ui/schema-validator.test.ts` |
| Module validation tests | `src/features/auto-ui/test/module-validation.test.ts` |
| Resource validation tests | `src/features/resource-version/validation-resource-version-data.test.ts` |
| UI matching tests | `src/modules/AutoUi/utils/validation-error.test.ts` |

Whenever a schema rule is changed, add or update a focused test before changing the implementation. Then run the relevant test file, the affected feature test suite, and finally the full project suite.
