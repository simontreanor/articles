# The Functional Blueprint: Teaching TypeScript to Speak F#

TypeScript has become the lingua franca of LLMs: ask a modern AI assistant to write backend services, frontend components, or shared domain models and it will almost certainly default to TypeScript-like code. The *style* of TypeScript the model chooses, however, has a significant impact on safety and maintainability.
If you come from an F# background, you already know a different way to write programs. F# emphasises immutable data, discriminated unions, exhaustive pattern matching, domain-driven modelling, and clear separation between pure logic and side effects, with the result that many classes of bugs are simply impossible to represent at the type level.
This article is a blueprint for bending TypeScript in that direction, using concrete patterns that can be encoded into LLM prompts and into your codebase. Each section covers a core F# idea and shows:
- How to emulate them in TypeScript.
- Where parity is strong and where it breaks down.
- How to guide LLMs so they *consistently* use those patterns.

***

## 1. Discriminated Unions and Tagged Unions

In F#, discriminated unions (DUs) are the primary way to model domain states explicitly, rather than relying on booleans, flags, or "nullable" fields.
### 1.1 F# discriminated unions

```fsharp
type Shape =
    | Circle    of radius: float
    | Rectangle of width: float * height: float
```

A `Shape` is either a `Circle` with a `radius`, or a `Rectangle` with `width` and `height`. Nothing else is allowed; the type is **closed**.
### 1.2 TypeScript tagged unions

TypeScript can emulate this with tagged unions (its own term for DUs).
```ts
type Shape =
  | { kind: "circle";    radius: number }
  | { kind: "rectangle"; width: number; height: number };
```

The `kind` field is the **discriminator**. It lets the compiler narrow within a `switch` or `if` chain.
#### Boilerplate and gaps

Compared to F#, there is more boilerplate:

- You must choose and repeat a tag field (`kind`, `type`, etc.).
- Adding a new case means changing the union declaration plus all `switch`es that depend on it.
For LLMs, you want to be explicit in your prompts:

> “Model domain states as tagged unions with a string `kind` discriminator instead of using booleans, nullable properties, or loosely shaped objects.”

Even this one constraint significantly shapes the output style.
***

## 2. Exhaustive Pattern Matching and `satisfies`

F#’s `match` is more than syntactic sugar: the compiler can tell you when you’ve forgotten to handle a case.
### 2.1 F# exhaustive pattern matching

```fsharp
let area shape =
    match shape with
    | Circle radius      -> System.Math.PI * radius * radius
    | Rectangle (w, h)   -> w * h
```

If you later add `| Triangle of base: float * height: float`, the compiler warns unless you update `area`.
### 2.2 TypeScript exhaustive matching with `satisfies never`

TypeScript's `switch` doesn't enforce exhaustiveness by default, but it can be approximated using the `never` type and the `satisfies` operator.
```ts
function getArea(shape: Shape): number {
  let area: number;
  switch (shape.kind) {
    case "circle":
      area = Math.PI * shape.radius ** 2;
      break;
    case "rectangle":
      area = shape.width * shape.height;
      break;
    default:
      area = shape satisfies never;
  }
  return area;
}
```

If you add a new `kind` to `Shape` but forget to handle it here, `shape satisfies never` causes a compile-time error.
This is the **consumption-site** exhaustiveness check.
### 2.3 Exhaustiveness at mapping sites

You can also enforce exhaustiveness where you define “total mappings”—for example, mapping each colour to a hex code.
```ts
type Color = "red" | "green" | "blue";

const ColorHex = {
  red:   "#f00",
  green: "#0f0",
  blue:  "#00f",
} satisfies Record<Color, string>; // Errors if any Color is missing
```

Here, `satisfies Record<Color, string>` ensures:

- Every `Color` key is present.
- The object’s literal types are preserved (you don’t widen to `Record<string, string>`).
This is the **definition-site** check: the object must cover all keys in the union.
### 2.4 Best practices for LLMs

Encode this in your prompts:

> "For every `switch` on a tagged union, add a `default` case that assigns `value satisfies never` to the result variable so that missing cases are compile-time errors."

> “For mappings over union keys, define objects that `satisfies Record<Union, T>` to guarantee all keys are covered.”

Both together give TypeScript something close to F#'s match experience.
***

## 3. Immutability, Records, and Structural Equality

In F#, records are immutable by default and compared structurally.
### 3.1 F# records

```fsharp
type Person = { Name: string; Age: int }

let p1 = { Name = "Alice"; Age = 30 }
// p1.Age <- 31 // compile-time error: record fields are immutable
```

Two records with identical fields and values are equal.
### 3.2 TypeScript records with `readonly`

TypeScript emulates records with object types and the `readonly` modifier:

```ts
type Person = {
  readonly name: string;
  readonly age:  number;
};

const p1: Person = { name: "Alice", age: 30 };

// p1.age = 31; // Error: Cannot assign to 'age' because it is a read-only property
```

For collections, use `ReadonlyArray<T>` or `readonly T[]`:

```ts
const people: ReadonlyArray<Person> = [
  { name: "Alice", age: 30 },
];
```

### 3.3 Structural equality gap

TypeScript uses reference equality for objects: `{ name: "Alice", age: 30 } === { name: "Alice", age: 30 }` is `false`. You need helper functions or libraries (`fast-deep-equal`, etc.) for structural equality.
### 3.4 Non-destructive updates

F# has `with` for persistent updates:

```fsharp
let older = { p1 with Age = p1.Age + 1 }
```

In TypeScript, object spread is close:

```ts
const older: Person = { ...p1, age: p1.age + 1 };
```

This is more verbose but conceptually similar.
### 3.5 Prompting for immutable style

When steering LLMs:

- “Make all domain object properties `readonly`.”
- “Use `ReadonlyArray` for lists and non-destructive updates via spread or pure functions.”
- “Do not mutate function arguments or shared state.”

***

## 4. Option/Result vs null/undefined

F# avoids `null` by design, preferring `Option<'a>` and `Result<'a, 'e>`.
### 4.1 F# `Option` and `Result`

```fsharp
let divide x y =
    if y = 0 then None
    else Some (x / y)

let tryParseInt input =
    match System.Int32.TryParse input with
    | true, value  -> Ok value
    | false, _     -> Error "Not a number"
```

Returning `Option` or `Result` forces callers to handle missing data and errors explicitly.
### 4.2 TypeScript `Option` emulation

TypeScript relies on unions and strict null checking:

```ts
function divide(x: number, y: number): number | undefined {
  return y === 0 ? undefined : x / y;
}
```

You can then use optional chaining and nullish coalescing:

```ts
const result = divide(10, 2) ?? 0;
```

For a more F#-like `Option`, you can define:

```ts
type Option<T> = { kind: "some"; value: T } | { kind: "none" };
```

…but in practice, `T | undefined` often wins on ergonomics.
### 4.3 TypeScript `Result` emulation

A tagged union is a natural fit:

```ts
type Result<T, E> =
  | { ok: true;  value: T }
  | { ok: false; error: E };

function divideSafe(x: number, y: number): Result<number, "DivideByZero"> {
  return y === 0
    ? { ok: false, error: "DivideByZero" }
    : { ok: true,  value: x / y };
}
```

Callers must inspect `ok`, mirroring F#’s `Result`. Libraries like `neverthrow` and `fp-ts` provide richer ecosystems around this idea.
### 4.4 LLM guidance

> “Do not throw exceptions for domain-level failures. Instead, return a `Result<T, E>` tagged union.”

> “Avoid `null`; prefer `undefined` in `T | undefined` unions or explicit `Option`/`Result` types.”

Both patterns bring TS closer to F#'s explicit failure semantics.
***

## 5. Pipelines and Composition

F#’s pipeline operator `|>` makes it easy to write left-to-right data flows.
### 5.1 F# pipelines

```fsharp
[1 .. 5]
|> List.map   (fun x -> x * 2)
|> List.filter(fun x -> x > 5)
```

The data flows left to right, making the intent clear.
### 5.2 TypeScript pipelines today

For arrays, chain methods:

```ts
const result =
  [1, 2, 3, 4, 5]
    .map(x => x * 2)
    .filter(x => x > 5);
```

For general functions, you can define a `pipe` helper:

```ts
type Unary<A, B> = (a: A) => B;

function pipe<A, B>(a: A, ab: Unary<A, B>): B;
function pipe<A, B, C>(a: A, ab: Unary<A, B>, bc: Unary<B, C>): C;
// …more overloads…
function pipe(a: unknown, ...fns: Unary<any, any>[]): unknown {
  return fns.reduce((v, f) => f(v), a);
}
```

Then:

```ts
const result = pipe(
  [1, 2, 3, 4, 5],
  xs => xs.map(x => x * 2),
  xs => xs.filter(x => x > 5),
);
```

TC39’s pipeline proposal may eventually reduce the need for this, but for now, a standard `pipe` helper is a good pattern to encourage from LLMs.
***

## 6. Units of Measure and Branded Types

Units of Measure are one of F#'s most distinctive features: you can't accidentally add metres and feet.
### 6.1 F# Units of Measure

```fsharp
[<Measure>] type m
[<Measure>] type ft

let distanceInMeters : float<m> = 10.0<m>
let distanceInFeet   : float<ft> = 32.8<ft>

// let total = distanceInMeters + distanceInFeet // compile-time error
```

### 6.2 TypeScript branded (nominal) types

In TypeScript’s structural system, `number` is `number`. To distinguish metres from feet, you “brand” them.
```ts
type Meters = number & { readonly __brand: "Meters" };
type Feet   = number & { readonly __brand: "Feet" };

const Meters = (n: number): Meters => n as Meters;
const Feet   = (n: number): Feet   => n as Feet;

const dMeters = Meters(10);
const dFeet   = Feet(32.8);

// let total: Meters = dMeters + dFeet; // Type error: Feet not assignable to Meters
```

You can generalise this using `unique symbol` for extra safety:
```ts
declare const __brand: unique symbol;

type Brand<T, TBrand> = T & { [__brand]: TBrand };

type UserId  = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
```

### 6.3 Why this matters (the “ID soup” problem)

LLMs naturally output things like:

```ts
function updateUser(id: string, orgId: string) { /* ... */ }
```

This is “ID soup”: any `string` can go anywhere. With branded types:
```ts
type UserId = Brand<string, "UserId">;
type OrgId  = Brand<string, "OrgId">;

function updateUser(id: UserId, orgId: OrgId) { /* ... */ }
```

You cannot accidentally swap the arguments without a compile error.
### 6.4 Arithmetic and JSON gaps

- Arithmetic on brands tends to “forget” the brand (because the compiler sees `number` operations), so you may need domain-specific helpers or re-branding.- When deserialising JSON, brands don’t exist at runtime; you must re-validate and brand at the boundary (e.g. using Zod or io-ts).
### 6.5 Prompting for branded types

> “Define branded (nominal) types for all domain identifiers (e.g. `UserId`, `OrderId`) and measurements. Functions that operate on those values must use the branded types so they cannot be confused.”

The result is F#-like domain safety at the type level.
***

## 7. Active Patterns, Matchers, and Prisms

Active Patterns in F# are an advanced feature that let you define *views* over data. They encode classification logic directly into pattern matching.
### 7.1 Multi-case active patterns in F#

```fsharp
let (|Email|Phone|Unknown|) (input: string) =
    if input.Contains("@")        then Email   input
    elif input |> Seq.forall System.Char.IsDigit
                                   then Phone  input
    else Unknown

let describe input =
    match input with
    | Email addr -> sprintf "Email: %s" addr
    | Phone num  -> sprintf "Phone: %s" num
    | Unknown    -> "Unknown contact method"
```

The pattern both **classifies** and **transforms** the data.
### 7.2 TypeScript “matcher function” pattern

TypeScript doesn’t have syntax to transform during the `switch` head, so you do it just before, via a matcher function that returns a tagged union.
```ts
type ContactMethod =
  | { type: "Email";   address: string }
  | { type: "Phone";   number:  string }
  | { type: "Unknown" };

const asContactMethod = (input: string): ContactMethod => {
  if (input.includes("@"))       return { type: "Email",   address: input };
  if (/^\d+$/.test(input))       return { type: "Phone",   number:  input };
  return { type: "Unknown" };
};

const method = asContactMethod("test@example.com");

switch (method.type) {
  case "Email":
    console.log(method.address);
    break;
  case "Phone":
    console.log(method.number);
    break;
  case "Unknown":
    break;
  default:
    method satisfies never;
}
```

The result is effectively a hand-built active pattern.
### 7.3 Partial active patterns and `T | undefined`

Partial active patterns—`(|Integer|_|)`—either match or they don’t. In TS, you can represent this via `T | undefined`:

```ts
const asInteger = (value: string): number | undefined => {
  const parsed = parseInt(value, 10);
  return Number.isNaN(parsed) ? undefined : parsed;
};

const n = asInteger("42") ?? 0;
```

This pairs nicely with nullish coalescing and avoids boolean blindness.
### 7.4 Prisms and lenses

In functional optics:

- A **Lens** focuses on a part that **always exists** (e.g. `User.name`).- A **Prism** focuses on a part that **might exist**, often inside a union (e.g. the `Circle` inside `Shape`).
Matcher functions are hand-built prisms: given arbitrary input, return either `{ type: "Email"; ... }` or `{ type: "Unknown" }`.
Asking LLMs to produce matchers rather than deeply nested `if` chains produces more F#-like structure.
### 7.5 Prompting for classification-first design

> “Don’t embed complex business logic directly in UI handlers or controllers. Instead, create ‘matcher’ functions that classify raw inputs into tagged unions, then handle those via exhaustive `switch` with `satisfies never`.”

This separates *classification* from *handling*, much like active patterns.
***

## 8. Boolean Blindness and Type-Level Literals

Boolean blindness is what happens when a `boolean` returns from a function but the *reason* behind `true` or `false` is not encoded anywhere.
### 8.1 The blind approach

```ts
function canAccess(user: User): boolean {
  // complex rules...
  return user.role === "admin" || user.isSubscriber;
}

if (canAccess(user)) {
  // But *why* was access granted? The type system has no idea.
}
```

You lose all nuance: “admin”, “subscriber”, “grace period”, etc.
### 8.2 Union-of-reasons result

Instead, return a discriminated union:

```ts
type Access =
  | { kind: "Allowed";   reason: "Admin" | "Subscriber" }
  | { kind: "Denied";    reason: "Unauthorized" | "Expired" };

function canAccess(user: User): Access {
  if (user.role === "admin")
    return { kind: "Allowed", reason: "Admin" };
  if (user.isSubscriber)
    return { kind: "Allowed", reason: "Subscriber" };
  return { kind: "Denied", reason: "Unauthorized" };
}
```

The caller can then distinguish why access was allowed or denied.
### 8.3 Template literal types as parameterised patterns

Template literal types let you encode information into string shapes.
```ts
type Px      = `${number}px`;
type Percent = `${number}%`;
type Length  = Px | Percent;

function setWidth(value: Length) {
  // `value` is guaranteed to be "Npx" or "N%"
}

// setWidth("100");   // Error
setWidth("100px");   // OK
setWidth("50%");     // OK
```

This is similar in spirit to parameterised active patterns like `Range(1, 10)`—you constrain shape at the type level before runtime.
You can also encode parameterised checks:

```ts
type MultipleOf<N extends number> = `MultipleOf_${N}`;

function checkFactor<N extends number>(
  n: number,
  factor: N
): MultipleOf<N> | "NoMatch" {
  return n % factor === 0
    ? (`MultipleOf_${factor}` as MultipleOf<N>)
    : "NoMatch";
}
```

The result type carries the factor it matched against.
### 8.4 Assertion functions

Where type-level encodings become unwieldy, assertion functions are the TS analogue of pattern-based narrowing:

```ts
type PositiveNumber = Brand<number, "PositiveNumber">;

function assertIsPositive(n: number): asserts n is PositiveNumber {
  if (n <= 0) {
    throw new Error("Not positive");
  }
}
```

After calling `assertIsPositive(value)`, the compiler treats `value` as `PositiveNumber` inside that scope.
### 8.5 Prompting away from booleans

> “Avoid returning `boolean` for domain decisions. Return a union of literal types or a tagged union that encodes the *reason* for the outcome.”

> “Use template literal types for domain-specific string formats: e.g. `ID_${string}`, `${number}px`, etc.”

Both rules reduce boolean blindness and push toward F#'s DU-heavy style.
***

## 9. Lists, Arrays, and Persistent Data Structures

F# lists are immutable linked lists; JS/TS arrays are mutable, indexed sequences.
### 9.1 F# lists

```fsharp
let numbers = [1; 2; 3; 4; 5]  // 'a list
let more    = 0 :: numbers     // prepend is O(1)
```

F# also has arrays and other structures, but lists are idiomatic for many recursive and functional patterns.
### 9.2 TypeScript arrays and `ReadonlyArray`

TypeScript’s default `T[]` is mutable. To approximate F# lists, use `ReadonlyArray<T>`:
```ts
const xs: ReadonlyArray<number> = [1, 2, 3];

// xs.push(4); // Error: Property 'push' does not exist on type 'readonly number[]'
const ys = [0, ...xs]; // creates a new array
```

### 9.3 Performance implications

The illusion of immutability through spreads has a cost: `[newItem, ...oldArray]` copies the entire array. For small lists (UI-level code) this is usually fine; for high-throughput data processing, consider:
- Structuring transforms as pipelines without excessive copying.
- Using libraries providing persistent data structures (e.g. Immutable.js, Mori, Immer in “copy-on-write” style).
### 9.4 Prompting for immutable collections

> “Use `ReadonlyArray<T>` and avoid mutating arrays in place (`push`, `splice`, `sort` in place). Use non-mutating methods (e.g. `map`, `filter`) or return new arrays.”

This preserves the functional, data-in/data-out character of F#'s list processing.
***

## 10. Modules vs Companion Objects and File Modules

In F#, you group related types and functions into modules, giving you names like `List.map` or `User.create`.
### 10.1 F# modules

```fsharp
module User =
    type T = { Id: int; Name: string }

    let create id name = { Id = id; Name = name }

    let validate (user: T) =
        user.Name.Length > 0
```

### 10.2 TypeScript’s “companion object” pattern

In modern TypeScript, the primary unit of modularity is the ES module (file). To mimic F# modules, define:
- A `type` for the data.
- A `const` (companion object) with the same name containing pure functions.
```ts
// User.ts

export type User = {
  readonly id:   UserId;
  readonly name: string;
};

export const User = {
  create(id: UserId, name: string): User {
    return { id, name };
  },

  validate(user: User): boolean {
    return user.name.length > 0;
  },
};
```

Usage:

```ts
const user = User.create(userId, "alice");
if (User.validate(user)) { /* ... */ }
```

This gives the `Module.function` style familiar from F#.
### 10.3 Namespaces (and why to avoid them)

TypeScript also has `namespace`:

```ts
namespace User {
  export type T = { /* ... */ };
  export const create = /* ... */;
}
```

This looks very F#-like, but is discouraged in modern code because it doesn’t play nicely with tree-shaking and bundlers compared to ES modules.
### 10.4 Public API control

F# uses `.fsi` signature files; TypeScript uses `export` and “barrel” files:

- Export only public entities from `index.ts`.
- Keep internal helpers unexported so they’re effectively module-private.
### 10.5 Prompting for module structure

> “Organise the code using a module-per-domain-type approach. For each domain entity, create a file defining the `type` and a `const` with the same name that contains pure functions. Do not use classes.”

This discourages anemic class patterns in favour of F#-like module design.
***

## 11. Consolidated Mapping: F# Features to TypeScript Patterns

| F# feature                 | TypeScript “functional” equivalent                              | Parity level / notes                         |
|---------------------------|------------------------------------------------------------------|----------------------------------------------|
| Discriminated Union       | Tagged union with `kind` discriminator                          | High; more boilerplate                        |
| Exhaustive pattern match  | `switch` + `satisfies never`                                     | High at usage and mapping sites               |
| Active Patterns           | Matcher functions, prisms returning tagged unions or `T \| undefined` | Medium; manual ceremony                       |
| Option                    | `T \| undefined` or `Option<T>` DU                                | High in practice                              |
| Result                    | `Result<T, E>` tagged union or `neverthrow`/`fp-ts`              | High with discipline                          |
| Records                   | `type`/`interface` + `readonly`                                 | High; no built-in structural equality         |
| Units of Measure          | Branded types (`Brand<T, Tag>`)                                  | Medium; arithmetic & JSON require helpers     |
| Lists (linked)           | `ReadonlyArray<T>` or persistent data structures library         | Low–medium; semantics differ                  |
| Pipelines                 | Chaining or `pipe(...)` helper                                   | Medium; pending pipeline operator             |
| Modules                   | File modules + companion objects                                 | High in practice                              |
| Null avoidance            | `strictNullChecks`, `undefined`, explicit unions                 | High with discipline                          |
| Boolean-free domain logic | Unions of literal reasons, template literal types                | High; requires design and conventions         |


***

## 12. A “Golden Prompt Library” for F#-Style TypeScript

These prompts can be combined into a reusable header that you paste into LLM sessions when generating code.

### 12.1 Structural modelling

> “Model domain data with tagged unions and immutable records: define `type` aliases with a string `kind` discriminator and `readonly` fields. Use `ReadonlyArray<T>` for collections.”

### 12.2 Exhaustiveness

> “All `switch` statements over tagged unions must be exhaustive. Add a `default` branch that uses `const _exhaustiveCheck: never = value satisfies never;` so that missing cases cause compile-time errors.”

### 12.3 Domain safety and branded types

> “Define branded (nominal) types for all domain identifiers (e.g. `UserId`, `OrgId`, `OrderId`) and critical measurements using `Brand<T, Tag>` or `number & { readonly __brand: ... }`. Functions must accept these branded types, not raw primitives.”

### 12.4 Error handling and options

> “Avoid `null` and throwing for normal domain errors. Use union types with `undefined` (`T | undefined`) for optional values and a `Result<T, E>` tagged union for operations that can fail.”

### 12.5 Logic structure and active patterns

> “Separate classification from handling: create matcher functions that transform raw inputs into tagged unions (like F# active patterns), and then handle them via exhaustive `switch` statements.”

### 12.6 Organisation and style

> “Organise code by domain module: one file per entity with a `type` for data and a `const` with the same name providing pure functions. Do not use classes; prefer pure functions, immutable data, and pipelines over mutation.”

Wording can be adjusted to suit the model, but together these constraints steer generated TypeScript toward a more functional, F#-inspired style.
***

TypeScript is flexible enough to host most of F#'s core ideas. Encoding these patterns into LLM prompts means that flexibility works in your favour rather than against it.
