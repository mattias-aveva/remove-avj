# Auto UI Validation

[AJV migration change log](AJV-MIGRATION-CHANGELOG.md)

## Purpose

Auto UI is a schema-driven form system. JSON Schemas describe module settings and are used to:

- generate form fields automatically
- provide default values
- handle objects, lists, dictionaries, credentials, and resources
- validate user input in the frontend
- display errors next to the correct setting

Frontend validation remains in place because users need immediate feedback while configuring modules.

## CSP Background

The hosting environment uses:

```text
script-src 'self'
```

and does not allow `unsafe-eval`.

AJV was replaced because runtime JSON Schema compilation may use dynamically generated code. The new implementation uses regular TypeScript logic and does not use `eval`, `new Function`, or runtime code generation.

## Architecture

```text
JSON Schema
    |
    v
PropertyInfo and form structure
    |
    v
Auto UI components
    |
    v
User settings
    |
    v
Internal TypeScript validator
    |
    v
Validation errors returned to the UI
```

### Schema handling

- `src/features/auto-ui/auto-ui-schema.ts`
- `src/features/auto-ui/module-schema.ts`
- `src/features/auto-ui/types-auto-ui.ts`

These files interpret JSON Schema and create `PropertyInfo`, containing type, default value, enum, structure, and UI metadata.

### Rendering

- `src/modules/AutoUi/AutoUi.vue`
- `src/modules/AutoUi/components/`

`AutoUi.vue` renders the appropriate input component for each `PropertyInfo`. Changes are emitted as `settingUpdate`. Validation is requested after a change with a delay of approximately 300 ms.

## Validation

### Shared validator

`src/features/auto-ui/schema-validator.ts`

The general validator returns internal errors in this format:

```ts
{
  instancePath: string
  keyword: string
  params: Record<string, unknown>
  message: string
}
```

Supported project-relevant JSON Schema functionality includes:

- types and nullable `null`
- `required`, `properties`, `items`
- `additionalProperties`
- `allOf`, `anyOf`, `oneOf`, `not`
- `enum` and `const`
- `pattern`, `minLength`, `maxLength`
- `minimum`, `maximum`
- `exclusiveMinimum`, `exclusiveMaximum`
- `multipleOf`, `minItems`, `maxItems`, `uniqueItems`
- local `$ref` references
- `if` / `then`
- `dependentRequired`
- `guid` and `uuid`
- `int32`

The `int32` format accepts the inclusive range:

```text
-2147483648 <= value <= 2147483647
```

Unresolved local `$ref` references return an explicit error and are not silently accepted.

### Auto UI adapter

`src/features/auto-ui/auto-ui-validation.ts`

`validateAutoUiSettings(...)` uses the shared validator and adapts errors to:

```ts
{
  setting: string
  errorMessage: string
}
```

The adapter preserves:

- conversion from `null` to `undefined` for non-nullable settings
- complete paths for nested `required` errors
- the GUID message `select a valid item from the list`
- handling of multiple errors
- logging and protection against internal validation failures

### Flow module validation

`src/features/auto-ui/module-setting-validation.ts`

This file handles module settings and Universal Connector parameters. It preserves logic for:

- merging `allOf` schemas
- SDK-based `commonSettings`
- dynamic Universal Connector schemas
- archived and incomplete resource versions
- validation of individual nested elements

### Resource Draft

`src/features/resource-version/validation-resource-version-data.ts`

This file validates data for these resource types:

- `Modbus`
- `DataTriggerDefinitions`
- `AvevaHistorian`

It preserves the existing error messages for invalid JSON, invalid input types, and schema errors:

```text
Failed to parse data. Expected JSON.
Unexpected format of resource data.
Resource contains error: ...
```

## Error Paths in the UI

Validation errors use slash-separated paths:

```text
simpleSettings/requiredStringSetting
listSettings/listOfObjects/0/value
dictionary/key/nestedField
```

`src/modules/AutoUi/utils/validation-error.ts` matches both exact paths and all actual child paths below a setting. Matching is path-boundary aware, so `settings/key` does not incorrectly match `settings/keyOther`.

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

## Tests

### Important test files

- `src/features/auto-ui/schema-validator.test.ts`
- `src/features/auto-ui/test/auto-ui-validation.test.ts`
- `src/features/auto-ui/test/auto-ui-validation.integration.test.ts`
- `src/features/auto-ui/test/module-validation.test.ts`
- `src/features/resource-version/validation-resource-version-data.test.ts`
- `src/modules/AutoUi/utils/validation-error.test.ts`

The `validation-test-module.schema.json` fixture represents the backend schema format and is used for realistic schema tests.

Tests cover:

- valid and invalid values
- required and nullable values
- nested paths
- enum, pattern, and length limits
- numeric limits and `int32`
- GUID/UUID
- objects, dictionaries, and arrays
- `$ref`, `oneOf`, `anyOf`, `not`, `if/then`, and `dependentRequired`
- Resource Draft parsing and schema errors
- UI matching of nested errors

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

Run the TypeScript check:

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

## Verification Status

The migration has been verified with:

- the full Jest test suite passing, with one existing skipped test
- Auto UI tests passing
- Resource Draft tests passing
- UI validation matching tests passing
- TypeScript passing
- ESLint passing for affected files
- the production build passing
- manual user testing completed successfully

Exact totals may change as tests are added. Use the latest command output as the source of truth.

## Dependencies

AJV and `ajv-draft-04` have been removed as direct dependencies from:

- `package.json`
- `package-lock.json`

Transitive AJV versions may remain in the lockfile if other tools require them. They must not be removed manually, because this could break build, lint, or editor packages.

## Maintenance Guide

Use these files for future changes:

| Change | File |
|---|---|
| General JSON Schema rule | `src/features/auto-ui/schema-validator.ts` |
| Auto UI errors and null handling | `src/features/auto-ui/auto-ui-validation.ts` |
| Module and Universal Connector logic | `src/features/auto-ui/module-setting-validation.ts` |
| Resource Draft behavior | `src/features/resource-version/validation-resource-version-data.ts` |
| UI error matching | `src/modules/AutoUi/utils/validation-error.ts` |

Always add a focused test when introducing a new schema rule or error path.

## Final Status

The functional AJV migration is complete. All production validation paths use the internal TypeScript validator, and the application does not require `unsafe-eval` for this validation functionality.

The implementation has been verified through automated tests, linting, TypeScript checks, production build, and successful manual user testing.
