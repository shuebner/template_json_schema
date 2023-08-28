# Summary

This repo serves as a basis for developing JSON schemas in a test-driven way.
The recommended development tool is VS Code.

# Folder structure and $ids

## Samples

Samples for both valid and invalid schema instances can be found in the "samples" folder.
They are meant to be consumed by both humans and automated tests.
They complement the `"examples"` in the schema files themselves.
Add/modify/delete samples when changing the schema.

Samples in the `samples/<some_name/` folders are supposed to be validated against a schema located in `schemas/<some_name>.schema.json`.

## Tests

The tests check the schema itself.
They are meant as a safeguard to ensure that the schemas are correct.
Correctness is checked by applying the schemas to the samples and checking that they are valid/invalid as expected.
The tests use `ajv` for this validation.

## Schemas

Contains all the schema files.
Subfolders can be created.

Schemas have an `$id` URI and can reference other schemas via `$ref` and a relative file path.
A referenced schema's `$id` URI **MUST** be relative to the referencing schema's URI in the same way that its file path is relative to the referencing schema's file path.
See also [the documentation](https://json-schema.org/understanding-json-schema/structuring.html#id1) and, maybe more understandably phrased [this SO answer](https://stackoverflow.com/a/63637901/1767703)

Visual Studio Code is lenient about this, but tools like `ajv` are not.

**Example**

file `somefolder/foo.schema.json`:
```json
{
    "$id": "mynamespace/foo.schema.json"
    ...
      "$ref": "bar/baz.schema.json"
}
```
file `somefolder/bar/baz.schema.json`:

```json
{
    // CORRECT: the id has the URI segment "bar" that corresponds to its relative file path
    "$id": "mynamespace/bar/baz.schema.json"

    // WRONG: the id has no corresponding relative URI segment
    // The solution is to either change the $id or move the file
    "$id": "mynamespace/bar.schema.json"
    "$id": "mynamespace/baz/bar.schema.json"
    ...
}
```

# Contribute

## Changing existing schemas

1. Change existing samples and add new valid/invalid samples as necessary.
2. Run the tests to see the expected tests fail for the expected reason. Usually some previously valid samples should not be valid anymore after the change.
3. Make the necessary changes to the schemas.
4. Run the tests again to check if schemas and samples are consistent again.


## Adding new schemas

When adding new schemas:

1. Add samples to help create and validate your schema.
   1. Add a valid sample in the `samples/<schemaName>/valid` folder
   1. Add one or many invalid samples in the `samples/<schemaName>/invalid` folder. Each invalid sample should ideally define the expected deviations to avoid false negatives. The deviations should be reflected in the file name as well.

1. Add a new schema file in `schemas`. You can copy an existing one to get the basics right.
   1. The file name should have the extension `.schema.json`.
   1. Set a proper value for the `$id` property. See [Folder structure and $ids](./README.md#Folder-structure-and-ids).

1. Run the tests as a quick sanity check.

# Peculiarities of tools consuming JSON schemas

From a JSON Schema point of view, schemas that extend a base schema could `$ref` the base schema and then declare `properties` themselves, potentially overwriting properties of the base schema (e. g. demanding the discriminator to be a constant).
This would have the advantage of preserving `additionalProperties: false`, which would lead to better generated TypeScript types with [json-schema-to-typescript
](https://github.com/bcherny/json-schema-to-typescript) (aka `json2ts`) without the unwanted `[k: string]: unknown`.
However, markdown generation with [jsonschema2md](https://github.com/adobe/jsonschema2md) would return errors.
That is why `allOf` must be used when employing both tools.

Somewhat related, from a JSON Schema point of view, schemas that extend a base schema with a discriminator could just set the `const` value for the discriminator, without repeating the `type: string` declaration.
The C# class generator of (NJsonSchema)[https://github.com/RicoSuter/NJsonSchema], however, needs the redeclaration to generate the discriminator property on the POCO with type `string`. Otherwise, it would generate it with type `object`.

From a JSON schema point of view, the `title` should be a short description of the property and its type.
jsonschema2md uses the `title` for its headings.
It should thus have a value and not be empty.
json2ts prefers the `title` as the basis for its type names.
We must thus choose an English value as the `title` that would also make a good type name.
In case there is an established German term for the property, the German term should be mentioned in the `description`.

In JSON Schema 2019-09 and later, using `$defs` instead of `definitions` is preferred.
However, NJsonSchema in version 10.8.0 only supports `definitions`, see [this issue](https://github.com/RicoSuter/NJsonSchema/issues/1536).
This means we must use `definitions`.
This is still [allowed by JSON Schema 2019-09](https://json-schema.org/draft/2019-09/release-notes.html#semi-incompatible-changes).

When nesting schemas (with `$id`s) and within the nested schema using a `$ref` that points to `#/definitions/something`, `ajv` and `jsonschema2md` will correctly resolve the `$ref` using the `definitions` property of the nested schema.
`NJsonSchema` and `jston2ts` will fail to resolve the reference.
Both of the latter seem only to be able to resolve relative file paths from within a nested schema.
When using a combination of tools on the same schema, this may necessitate putting referenced schemas in their own files, even if they are only referenced from a single place (a subschema).