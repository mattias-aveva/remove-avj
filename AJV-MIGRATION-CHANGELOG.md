# AJV Migration Change Log

## Summary

The frontend JSON Schema validation was migrated from AJV to an internal TypeScript validator to remove runtime schema compilation and avoid the need for `unsafe-eval` in the Content Security Policy.

Frontend validation remains in place so users continue to receive immediate validation feedback while configuring modules, credentials, and resource data.

## Motivation

The hosting environment enforces a CSP policy using:

```text
script-src 'self'
```

without `unsafe-eval`. AJV can compile schemas at runtime using generated functions, which is incompatible with this policy. Because frontend validation is required for user guidance, the validation logic was moved to an internal implementation instead of being removed or moved exclusively to the backend.

## Implemented Changes

### Internal schema validator

Created:

- `src/features/auto-ui/schema-validator.ts`

The validator is implemented in TypeScript and does not use:

- `eval`
- `new Function`
- runtime code generation
- an external validation library

The validator returns an internal error structure containing:

```ts
{
  instancePath: string
  keyword: string
  params: Record<string, unknown>
  message: string
}
```

Supported schema functionality includes:

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
- local `$ref` references
- `if` / `then`
- `dependentRequired`
- `guid` and `uuid` formats
- `int32` range validation

The `int32` format validates the inclusive range:

```text
-2147483648 <= value <= 2147483647
```

Unresolved local `$ref` references now produce an explicit validation error instead of silently accepting the value.

### Auto UI validation

Updated:

- `src/features/auto-ui/auto-ui-validation.ts`

`validateAutoUiSettings(...)` now uses the internal schema validator instead of AJV.

The existing Auto UI behavior was preserved, including:

- `null` to `undefined` adaptation for non-nullable properties
- conversion to `AutoUiValidationError`
- GUID-specific user guidance:

```text
select a valid item from the list
```

- collection of all validation errors
- graceful handling of validator exceptions
- full nested paths for required-property errors

For example:

```text
commonSettings/foo/bar
```

is retained as the setting path instead of being reduced to only `bar`.

### Flow module validation

Updated:

- `src/features/auto-ui/module-setting-validation.ts`

The following validation paths now use the internal validator:

- `validateFlowModuleSettings(...)`
- `validateElementSettingsProperty(...)`

Existing module-specific behavior was retained, including:

- `allOf` schema merging using `json-schema-merge-allof`
- replacement of `commonSettings` with the SDK schema
- dynamic Universal Connector parameter schemas
- resource-version checks for archived and incomplete resources
- existing Data Mapper and Split module exceptions
- Auto UI error mapping

The nested element-property lookup was also corrected. After `json-schema-merge-allof` processes a schema, the result may no longer contain `allOf`. The lookup now searches through:

- `properties`
- `allOf`
- `oneOf`
- `anyOf`
- nested `items`

This allows validation errors in nested list and object settings to be detected.

### Resource Draft validation

Updated:

- `src/features/resource-version/validation-resource-version-data.ts`

Resource Draft validation now uses the internal validator instead of AJV.

The existing public behavior was retained:

- resource types without schema validation are skipped
- JSON strings are parsed before validation
- invalid JSON returns:

```text
Failed to parse data. Expected JSON.
```

- unsupported input types return:

```text
Unexpected format of resource data.
```

- validation errors are returned using:

```text
Resource contains error: ...
```

The error selection was made safe when validation produces only a `not` error.

### UI validation error matching

Updated:

- `src/modules/AutoUi/utils/validation-error.ts`

Nested validation errors are now recognized for arbitrary child paths, not only the former special case matching `/<index>/value`.

The matching behavior now supports:

- exact setting paths
- nested object paths
- dictionary paths
- array item paths
- deeply nested child paths

Sibling properties with similar prefixes are not treated as matches.

## Tests Added or Updated

### Auto UI integration and parity tests

Added or updated:

- `src/features/auto-ui/test/auto-ui-validation.integration.test.ts`
- `src/features/auto-ui/schema-validator.test.ts`

Coverage includes:

- valid settings
- required properties
- nested required paths
- nullable values
- string length limits
- regular expression patterns
- numeric minimum and maximum values
- enum values
- GUID values
- additional properties
- nested array objects
- `oneOf`
- `int32` boundaries
- unresolved `$ref`
- `not`-only validation failures

### Module validation tests

Updated:

- `src/features/auto-ui/test/module-validation.test.ts`

Coverage includes:

- existing module validation behavior
- Universal Connector validation
- nested element-property validation after `allOf` merging

### Resource validation tests

Added:

- `src/features/resource-version/validation-resource-version-data.test.ts`

Coverage includes:

- resource types without schema validation
- invalid JSON input
- unsupported input types
- valid and invalid Modbus data
- Modbus conditional `length` validation for string tags
- DataTrigger minimum item count
- valid DataTrigger definitions
- value requirements for value-based conditions
- `dependentRequired`
- valid and invalid Aveva Historian data

### UI error matching tests

Added:

- `src/modules/AutoUi/utils/validation-error.test.ts`

Coverage includes:

- exact paths
- nested child paths
- array item paths
- dictionary child errors
- sibling-path exclusion

## Verification Results

The following checks were run successfully after the migration changes:

- Full Jest test suite: `955` tests passed, `1` skipped
- Auto UI test suite: `62` tests passed
- Resource validation tests: `12` tests passed
- UI validation matching tests: `6` tests passed
- Schema validator regression tests: `3` tests passed
- TypeScript check: `npx tsc --noEmit`
- ESLint for affected files
- Production build: `npm run build`

A temporary AJV differential test was also used during development. It compared representative Modbus, DataTrigger, and Aveva Historian cases against the previous AJV implementation. All `9/9` representative cases matched for validity and first error keyword. The temporary test was removed after the comparison.

## Current Dependency Status

The functional migration is complete, but AJV cleanup remains a separate final step.

The following legacy code still exists and should be removed after user testing and final approval:

- AJV imports and the unused `ajvWithAddons()` helper in `src/features/auto-ui/auto-ui-validation.ts`
- AJV-specific tests and mocks in `src/features/auto-ui/test/auto-ui-validation.test.ts`
- `ajv` and `ajv-draft-04` dependencies in `package.json`
- corresponding entries in `package-lock.json`

The final dependency cleanup should only be performed after confirming that no production or test code still requires AJV.

## Remaining Validation Work

The following manual verification remains important:

1. Add the Validation Test module to a flow in the target environment.
2. Confirm that invalid initial settings display validation errors immediately.
3. Confirm that errors remain visible for nested objects, dictionaries, and arrays.
4. Confirm that editing a value clears or updates the corresponding error.
5. Confirm that Resource Draft validation behaves correctly for the supported resource types.
6. Verify the generated production bundle and browser CSP behavior.

## Recommended Final Cleanup Sequence

1. Complete manual user testing.
2. Remove the unused AJV helper and imports.
3. Update the AJV-specific unit tests.
4. Remove `ajv` and `ajv-draft-04` from `package.json`.
5. Regenerate or update `package-lock.json`.
6. Search for remaining AJV references:

```bash
rg "ajv|Ajv|unsafe-eval" src package.json package-lock.json
```

7. Run the complete test suite.
8. Run lint and TypeScript checks.
9. Run the production build.
10. Verify the CSP policy in the deployed browser environment.

## Final Status

The runtime validation paths have been migrated from AJV to the internal TypeScript validator while preserving frontend validation and user-facing error behavior for the tested schema functionality.

The implementation is functionally verified, but the final dependency and dead-code cleanup should follow successful manual user testing.
