# Pretty Good JSON Parser (PGJP)

A robust, error-tolerant JSON parser with schema validation designed
specifically for handling partial, incomplete, or malformed JSON from AI tools,
streaming contexts, and other unreliable sources.

[![npm version](https://img.shields.io/npm/v/enhanced-json-parser.svg)](https://www.npmjs.com/package/enhanced-json-parser)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Motivation

When working with AI tools like GPT-4, Claude, or other large language models,
they often generate JSON responses that might be:

- Incomplete due to token limits or streaming interruptions
- Missing closing brackets or quotes
- Including invalid syntax like unquoted keys or trailing commas
- Mixing string formats (single quotes, double quotes)

Standard `JSON.parse()` fails on these inputs, requiring complex error handling.
This library provides a graceful solution, attempting to extract as much
meaningful data as possible from malformed JSON, while also supporting schema
validation and type coercion through Zod.

## Features

- ✅ Parses incomplete or malformed JSON without throwing errors
- ✅ Schema validation and type coercion with
  [Zod](https://github.com/colinhacks/zod)
- ✅ Type-safe with full TypeScript support
- ✅ Handles common AI output issues (missing brackets, invalid syntax)
- ✅ Supports nested objects and arrays
- ✅ Automatic type conversion (string→number, string→boolean, etc.)
- ✅ Compatible with standard JSON.parse() for valid JSON
- ✅ Minimal dependencies and lightweight

## Installation

```bash
npm install enhanced-json-parser
# or
yarn add enhanced-json-parser
# or
pnpm add enhanced-json-parser
```

## Basic Usage

```typescript
import { parse } from 'enhanced-json-parser';

// Parse valid JSON - works like standard JSON.parse()
const validResult = parse('{"name": "John", "age": 30}');
console.log(validResult); // { name: 'John', age: 30 }

// Parse incomplete JSON (missing closing bracket)
const incompleteResult = parse('{"name": "John", "age": 30');
console.log(incompleteResult); // { name: 'John', age: 30 }

// Handle malformed JSON (unquoted keys, missing commas)
const malformedResult = parse(`{
  name: John Doe
  age: 30
}`);
console.log(malformedResult); // { name: 'John Doe', age: 30 }
```

## Schema Validation with Zod

The parser supports schema validation and type coercion using Zod schemas:

```typescript
import { parse } from 'enhanced-json-parser';
import { z } from 'zod';

// Define a schema
const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
  email: z.string().email().optional(),
  isActive: z.boolean().default(true),
});

type User = z.infer<typeof UserSchema>;

// Parse and validate incomplete JSON against schema
const json = `{
  "name": "John Doe",
  "age": "30" // String instead of number
}`;

// String "30" is automatically converted to number 30
const user = parse<User>(json, UserSchema);
console.log(user);
// {
//   name: "John Doe",
//   age: 30,
//   isActive: true  // Default value from schema
// }
```

## Advanced Features

### Handling Partial Objects from AI Responses

When receiving partial JSON from AI tools:

```typescript
import { parse } from 'enhanced-json-parser';
import { z } from 'zod';

const EmailSchema = z.object({
  subject: z.string(),
  body: z.string(),
  recipients: z.array(z.string()),
  priority: z.enum(['high', 'normal', 'low']).default('normal'),
});

type Email = z.infer<typeof EmailSchema>;

// AI-generated partial email (incomplete recipients array, missing priority)
const aiResponse = `{
  "subject": "Meeting Reminder",
  "body": "Don't forget our meeting tomorrow at 2pm",
  "recipients": ["john@example.com", "jane@example.com`; // Cut off mid array

const email = parse<Email>(aiResponse, EmailSchema);
console.log(email);
// {
//   subject: "Meeting Reminder",
//   body: "Don't forget our meeting tomorrow at 2pm",
//   recipients: ["john@example.com", "jane@example.com"],
//   priority: "normal" // Default from schema
// }
```

### Handling Complex Nested Structures

The parser handles deep nesting and recursively validates against schemas:

```typescript
import { parse } from 'enhanced-json-parser';
import { z } from 'zod';

const ApiResponseSchema = z.object({
  status: z.number(),
  data: z.object({
    users: z.array(
      z.object({
        id: z.number(),
        profile: z.object({
          name: z.string(),
          email: z.string().email().nullable(),
        }),
      }),
    ),
    pagination: z.object({
      page: z.number(),
      totalPages: z.number(),
    }),
  }),
});

// Missing closing brackets, malformed structure
const json = `{
  "status": 200,
  "data": {
    "users": [
      { "id": 1, "profile": { "name": "Alice", "email": "alice@example.com" }},
      { "id": 2, "profile": { "name": "Bob", "email": null }},
      { "id": 3, "profile": { "name": "Charlie" }}
    ],
    "pagination": {
      "page": 1
`;

const response = parse(json, ApiResponseSchema);
console.log(response);
// Successfully parsed with totalPages defaulting to 0
```

### Auto Type Coercion

The parser automatically coerces types to match schema requirements:

```typescript
import { parse } from 'enhanced-json-parser';
import { z } from 'zod';

const FormSchema = z.object({
  userId: z.number(),
  active: z.boolean(),
  score: z.number(),
});

// Form data with string types (common in HTML forms)
const formData = `{
  "userId": "1001",
  "active": "true",
  "score": "invalid"
}`;

const result = parse(formData, FormSchema);
console.log(result);
// {
//   userId: 1001, // String converted to number
//   active: true, // String "true" converted to boolean
//   score: 0     // Invalid number string coerced to 0
// }
```

### Discriminated Unions

```typescript
import { parse } from 'enhanced-json-parser';
import { z } from 'zod';

const Shape = z.discriminatedUnion('type', [
  z.object({ type: z.literal('circle'), radius: z.number() }),
  z.object({ type: z.literal('square'), sideLength: z.number() }),
  z.object({
    type: z.literal('rectangle'),
    width: z.number(),
    height: z.number(),
  }),
]);

// Parse a circle with incomplete JSON
const circleJson = `{ "type": "circle", "radius": 5`;
const circle = parse(circleJson, Shape);
console.log(circle); // { type: 'circle', radius: 5 }

// Parse a square with broken JSON and missing fields
const squareJson = `{ type: "square" }`;
const square = parse(squareJson, Shape);
console.log(square); // { type: 'square', sideLength: 0 }
```

## API Reference

### `parse<T>(s: string | undefined | null, schema?: z.ZodType<T>): T`

Parses a JSON string with error tolerance and optional schema validation.

- `s`: String to parse, or `undefined`/`null`
- `schema`: Optional Zod schema for validation and type coercion
- Returns: Parsed object of type T

### `parse.lastParseRemaining`

Contains any remaining unparsed text from the last parsing operation.

### `parse.onExtraToken`

Handler for extra tokens in the parsed JSON. Defaults to throwing an error, but
can be replaced with a custom handler.

### `parse.coerceIfNeeded<T>(value: any, schema?: z.ZodType<T>): T`

Utility function to coerce an existing JavaScript object to match a Zod schema.

## Credits

This library is built on top of
[best-effort-json-parser](https://github.com/beenotung/best-effort-json-parser)
by [Beeno Tung](https://github.com/beenotung), with significant enhancements for
schema validation, type coercion, and handling of complex data structures.

## License

MIT License
