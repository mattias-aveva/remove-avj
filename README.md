# Changelog: AJV Migration

## Summary

Frontend JSON Schema validation was migrated from AJV to an internal TypeScript validator to comply with the CSP policy `script-src 'self'` without `unsafe-eval`.

Frontend validation and immediate user feedback remain available.

## Changes

### Internal validator

Created:

- `src/features/auto-ui/schema-validator.ts`

The validator uses no `eval`, `new Function`, runtime code generation, or external validation library.

Supported project-relevant JSON Schema functionality includes:

- types and nullable `null`
- `required`, `properties`, and `items`
- `allOf`, `anyOf`, `oneOf`, and `not`
- `$ref`, `enum`, and `const`
- `pattern`, `minLength`, and `maxLength`
- `minimum`, `maximum`, `exclusiveMinimum`, and `exclusiveMaximum`
- `multipleOf`, `minItems`, `maxItems`, and `uniqueItems`
- `if/then` and `dependentRequired`
- `guid`, `uuid`, and `int32`
- objects, arrays, and `additionalProperties`

The `int32` format validates the inclusive range:

```text
-2147483648 <= value <= 2147483647
```

Unresolved local `$ref` references now return an explicit validation error instead of being silently accepted.

### Migrated validation flows

The internal validator is now used by:

- Auto UI settings: `src/features/auto-ui/auto-ui-validation.ts`
- Flow Module settings: `src/features/auto-ui/module-setting-validation.ts`
- Resource Draft data: `src/features/resource-version/validation-resource-version-data.ts`

Existing behavior was preserved, including:

- `null` handling
- Universal Connector parameters
- SDK-based `commonSettings`
- resource-version checks
- nested element validation
- the GUID message `select a valid item from the list`
- the existing `AutoUiValidationError` format
- Resource Draft error messages

### Error-path handling

Nested `required` errors now keep their complete path, for example:

```text
simpleSettings/requiredStringSetting
```

Nested object, dictionary, and array errors are correctly matched to their UI components. Similar sibling paths are not matched accidentally.

### Dependency cleanup

Removed as direct dependencies:

- `ajv`
- `ajv-draft-04`

AJV may still exist transitively in `package-lock.json` when required by other tools. Those transitive dependencies are intentionally retained.

## Tests

Added or updated tests cover:

- Auto UI validation against a backend-style schema
- Flow Module and Universal Connector validation
- Resource Draft validation for Modbus, DataTrigger, and Aveva Historian
- `$ref`, `if/then`, `dependentRequired`, and `not`
- nested required, object, dictionary, and array errors
- `int32` boundaries
- UI validation-error matching
- JSON parsing and unsupported input types

## Verification

Completed checks:

- Full test suite: `950` tests passed, `1` skipped
- ESLint: passed
- TypeScript: passed
- Production build: passed
- Manual user testing: completed successfully
- `package-lock.json`: valid and package versions preserved

## Status

The functional AJV migration is complete. All production validation flows use the internal TypeScript validator and do not require `unsafe-eval` for this validation functionality.

The implementation is verified by automated tests, linting, TypeScript checks, a production build, and successful manual user testing. The change is ready for pull request review.
