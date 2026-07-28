# Odyssey Intro to GraphQL with TypeScript: Project Structure Guide

This document explains how this particular GraphQL server is assembled. It is
intended to be both a project map and reusable context that you can give to
ChatGPT when you want a tutor to explain or extend the server.

## 1. What this project is

This is a small Apollo Server application written in TypeScript. It exposes a
GraphQL API for reading and creating intergalactic property listings. The
GraphQL server does not own a database. Its resolvers call an existing REST API
through an Apollo `RESTDataSource`.

At a high level:

```text
GraphQL client
    |
    | GraphQL query or mutation over HTTP
    v
Apollo Server
    |
    +-- validates the operation against schema.graphql
    +-- calls the matching resolver
    +-- gives the resolver a request context
    |
    v
ListingAPI (RESTDataSource)
    |
    | HTTP GET or POST
    v
External listings REST service
```

The important architectural idea is that each layer has a different job:

- The schema describes what GraphQL clients are allowed to ask for.
- Resolvers explain how to obtain or compute fields in that schema.
- The context exposes request-scoped dependencies to the resolvers.
- The data source contains the details of communicating with the REST API.
- The entry point creates those parts and starts the HTTP server.

## 2. Project tree

Generated and dependency directories are shown for context even though they
are not source files that should normally be edited.

```text
odyssey-intro-typescript/
|-- src/
|   |-- datasources/
|   |   `-- listing-api.ts    # Wrapper around the external REST API
|   |-- context.ts            # Type of the Apollo request context
|   |-- graphql.d.ts          # TypeScript declaration for *.graphql imports
|   |-- helpers.ts            # Small resolver helper functions
|   |-- index.ts              # Composition root and server entry point
|   |-- resolvers.ts          # Query, mutation, and field resolver functions
|   |-- schema.graphql        # GraphQL schema written in SDL
|   `-- types.ts              # Generated TypeScript types; do not hand-edit
|-- codegen.ts                # GraphQL Code Generator configuration
|-- package.json              # Dependencies and npm scripts
|-- tsconfig.json             # TypeScript compiler configuration
|-- README.md                 # Original Apollo Odyssey course README
|-- PROJECT_STRUCTURE.md      # This learning guide
|-- dist/                     # Generated JavaScript build output (ignored)
`-- node_modules/             # Installed packages (ignored)
```

## 3. File-by-file walkthrough

### `src/schema.graphql`: the public API contract

The schema is written in GraphQL Schema Definition Language (SDL). It declares
three operation entry points:

```graphql
type Query {
  featuredListings: [Listing!]!
  listing(id: ID!): Listing
}

type Mutation {
  createListing(input: CreateListingInput!): CreateListingResponse!
}
```

It also describes the shapes used by those operations:

- `Listing` is the main domain object.
- `Amenity` is a nested object belonging to a listing.
- `CreateListingInput` groups the arguments accepted by the mutation.
- `CreateListingResponse` gives the mutation a consistent payload containing
  status information and an optional listing.

GraphQL nullability is worth reading carefully:

- `Listing!` means an individual listing cannot be `null`.
- `[Listing!]` means the list may be `null`, but its items may not be `null`.
- `[Listing!]!` means neither the list nor any item in it may be `null`.
- `listing(id: ID!): Listing` requires `id`, but permits no listing to be found.

The schema is the source of truth for code generation. When it changes,
`src/types.ts` needs to be regenerated.

### `src/index.ts`: the composition root

This file wires the application together:

1. It reads `schema.graphql` from disk.
2. `gql(...)` parses the SDL into a GraphQL document.
3. It creates `ApolloServer` with `typeDefs` and `resolvers`.
4. `startStandaloneServer` creates and starts a simple HTTP server.
5. The `context` function creates a new `ListingAPI` for each GraphQL request.
6. Apollo's cache is passed into that data source.

The relevant composition is equivalent to:

```ts
const server = new ApolloServer({ typeDefs, resolvers });

await startStandaloneServer(server, {
  context: async () => ({
    dataSources: {
      listingAPI: new ListingAPI({ cache: server.cache }),
    },
  }),
});
```

This is called the composition root because it is where otherwise separate
parts are instantiated and connected. The standalone server uses port `4000`
by default; this project does not currently supply a different `listen` option.

### `src/context.ts`: request-scoped dependencies

Apollo passes a context object to every resolver involved in one GraphQL
operation. This project gives the context one dependency:

```ts
export type DataSourceContext = {
  dataSources: {
    listingAPI: ListingAPI;
  };
};
```

That type gives resolvers compile-time knowledge of
`context.dataSources.listingAPI`. Creating the data source inside the context
function also keeps per-request memoization and request-specific state from
leaking between users.

In a larger server, context commonly also contains the authenticated user,
authorization helpers, database clients, loggers, and request metadata.

### `src/resolvers.ts`: how GraphQL fields get their values

A resolver is a function that supplies a value for a schema field. The
resolver map is organized by GraphQL type and then field name:

```text
resolvers
|-- Query
|   |-- featuredListings
|   `-- listing
|-- Listing
|   `-- amenities
`-- Mutation
    `-- createListing
```

Every resolver can receive four positional parameters:

```ts
(parent, args, context, info) => result
```

- `parent` is the value returned by the resolver one level above it.
- `args` contains the GraphQL field arguments.
- `context` is the request-scoped object created in `index.ts`.
- `info` contains execution details such as the selected fields.

The current query resolvers delegate directly to the REST data source:

- `Query.featuredListings` calls `getFeaturedListings()`.
- `Query.listing` passes `args.id` to `getListing(id)`.

`Listing.amenities` is a field resolver. It first inspects the listing returned
by the REST API. If the amenities appear complete, it returns them. Otherwise,
it makes a follow-up REST request using the parent listing's `id`. Apollo only
runs this resolver when the client selects the `amenities` field.

`Mutation.createListing` calls the data source and translates success or
failure into the `CreateListingResponse` shape declared by the schema.

Fields without custom resolvers use GraphQL's default field resolver. For
example, `Listing.title` is normally read from the `title` property of the
listing object returned by the REST API.

### `src/datasources/listing-api.ts`: the REST integration

`ListingAPI` extends Apollo's `RESTDataSource`. It owns the upstream base URL
and maps application methods to REST operations:

| TypeScript method | HTTP operation | Purpose |
| --- | --- | --- |
| `getFeaturedListings()` | `GET featured-listings` | Load homepage listings |
| `getListing(id)` | `GET listings/:id` | Load one listing |
| `getAmenities(id)` | `GET listings/:id/amenities` | Load full amenity data |
| `createListing(input)` | `POST listings` | Create a listing |

Keeping URLs and HTTP details here prevents the GraphQL resolvers from being
tightly coupled to the REST service. A resolver asks for domain data; the data
source decides how that data is fetched.

### `src/helpers.ts`: reusable resolver logic

`validateFullAmenities` checks whether an amenities array contains an item
with a `name` property. The `Listing.amenities` resolver uses this to decide
whether it already has expanded amenity objects or only partial amenity data.

One detail to notice while studying: its current use of `Array.some()` means
the array is considered complete when at least one item has `name`. If the
intent were to guarantee that every amenity is complete, `Array.every()` would
express that stricter rule.

### `codegen.ts` and `src/types.ts`: schema-to-TypeScript generation

GraphQL Code Generator reads `src/schema.graphql` and produces `src/types.ts`
using these plugins:

- `typescript` generates TypeScript representations of schema types.
- `typescript-resolvers` generates resolver signatures.

The config also connects the resolver context to `DataSourceContext`:

```ts
config: {
  contextType: "./context#DataSourceContext",
}
```

This is why importing `Resolvers` in `resolvers.ts` type-checks resolver names,
arguments, return values, parents, and context access. `src/types.ts` is ignored
by Git because it can be recreated; it should not be edited by hand.

### `src/graphql.d.ts`: a module declaration

This declaration tells TypeScript what a `*.graphql` module would contain if a
GraphQL file were imported directly. The current `index.ts` reads the schema
with `readFileSync` instead, so this declaration is not central to the active
startup path, but it supports an alternative import-based setup.

### `tsconfig.json`: compilation rules

The TypeScript compiler:

- reads source from `src`;
- writes CommonJS JavaScript and declarations to `dist`;
- targets modern JavaScript (`esnext`);
- checks implicit `any`, missing returns, fallthrough cases, and filename case;
- excludes generated build output and `codegen.ts` from the application build.

## 4. What happens during a request

Consider this operation:

```graphql
query GetListing($listingId: ID!) {
  listing(id: $listingId) {
    id
    title
    amenities {
      id
      name
    }
  }
}
```

With variables:

```json
{
  "listingId": "listing-1"
}
```

The execution path is:

```text
1. Apollo receives and parses the operation.
2. Apollo validates it against schema.graphql.
3. index.ts's context function creates ListingAPI for this request.
4. Query.listing receives { id: "listing-1" } in args.
5. Query.listing calls ListingAPI.getListing("listing-1").
6. ListingAPI performs GET listings/listing-1.
7. Apollo reads id and title from the returned object.
8. Apollo runs Listing.amenities because amenities was selected.
9. The field resolver returns existing full amenities or fetches them.
10. Apollo shapes the final JSON to exactly match the selection set.
```

This last point is a defining GraphQL feature: even if the REST service returns
many properties, the client receives only the fields selected in its GraphQL
operation.

## 5. Query and mutation examples

Fetch featured listings:

```graphql
query GetFeaturedListings {
  featuredListings {
    id
    title
    costPerNight
  }
}
```

Create a listing:

```graphql
mutation CreateListing($input: CreateListingInput!) {
  createListing(input: $input) {
    code
    success
    message
    listing {
      id
      title
    }
  }
}
```

Example variables:

```json
{
  "input": {
    "title": "Moon Base Retreat",
    "description": "A quiet stay with a view of Earth.",
    "numOfBeds": 2,
    "costPerNight": 250.5,
    "closedForBookings": false,
    "amenities": ["amenity-1", "amenity-2"]
  }
}
```

## 6. Commands and generated artifacts

Install dependencies:

```powershell
npm install
```

Start development mode:

```powershell
npm run dev
```

This runs two processes concurrently:

- `ts-node-dev` executes TypeScript and restarts when source or schema files
  change.
- GraphQL Code Generator watches `schema.graphql` and regenerates `types.ts`.

Generate types once without keeping a watcher alive:

```powershell
npx graphql-codegen
```

Compile TypeScript:

```powershell
npm run compile
```

Run tests:

```powershell
npm test
```

There are currently no test files in `src`, although Jest and `ts-jest` are
configured in `package.json`.

## 7. Current project limitations to understand

These are useful learning observations about the repository as it exists, not
requirements for completing the Odyssey tutorial.

1. **The production build does not copy the schema.** `tsc` creates
   `dist/index.js`, where `__dirname` points to `dist`, but it does not copy
   `src/schema.graphql` to `dist/schema.graphql`. Consequently, the current
   compiled entry point fails to start with `ENOENT`. A production-ready build
   should copy the schema, bundle it, import it through an appropriate loader,
   or resolve a stable source path.
2. **Generated types must exist before compilation.** `src/types.ts` is ignored
   by Git, while authored files import it. A fresh checkout needs code
   generation before a standalone compile. Development mode starts generation
   and the server concurrently, which can also create a first-run race.
3. **The server depends on an external REST service.** Queries can fail even
   when Apollo is healthy if the configured upstream service is unavailable or
   its response shape changes.
4. **Mutation error handling assumes a particular upstream error shape.** The
   catch block reads `err.extensions.response.body`. Production code should
   narrow unknown errors and avoid exposing sensitive upstream details.
5. **Nested REST calls can become an N+1 pattern.** Asking for amenities on a
   list of many incomplete listings may cause one follow-up request per
   listing. Caching, batching, or a bulk REST endpoint can address this at
   scale.
6. **Common production concerns are intentionally absent.** This tutorial does
   not yet include authentication, authorization, environment-based config,
   structured logging, health checks, rate limiting, persisted storage, or
   resolver tests.

## 8. A practical learning path

Study or change the server in this order:

1. Read `schema.graphql` and predict which operations are valid.
2. Match every root schema field to its resolver in `resolvers.ts`.
3. Trace each resolver call into `ListingAPI` and its REST endpoint.
4. Follow how `index.ts` supplies the data source through context.
5. Change one schema field and inspect the generated change in `types.ts`.
6. Add a resolver test with a fake or mocked `ListingAPI`.
7. Fix the production schema-copy issue and verify `dist/index.js` starts.
8. Add authentication data to context and enforce a rule in the mutation.
9. Investigate batching if nested amenities produce many REST calls.

Good questions to ask while learning:

- Why is the schema not enough by itself?
- When is a custom field resolver required?
- Why put `ListingAPI` in context instead of constructing it in each resolver?
- What is request-scoped state, and why does its lifetime matter?
- How do GraphQL errors differ from mutation payload errors?
- Which layer should validate input or enforce authorization?
- How would this design change if data came from a database instead of REST?

## 9. Reusable ChatGPT tutor prompt

Copy the prompt below into a new ChatGPT conversation. You can append a
specific topic or exercise after it.

```text
Act as my TypeScript and GraphQL tutor. I am studying a small Apollo Server 5
project and want to understand the reasoning behind its architecture, not just
copy code.

Project architecture:
- src/schema.graphql is the SDL contract. It defines Query.featuredListings,
  Query.listing(id), Mutation.createListing(input), Listing, Amenity,
  CreateListingInput, and CreateListingResponse.
- src/index.ts reads the SDL with readFileSync, parses it with gql, creates an
  ApolloServer from typeDefs and resolvers, and starts it with
  startStandaloneServer.
- The context function creates one ListingAPI RESTDataSource per GraphQL
  request and passes Apollo's cache to it.
- src/context.ts defines DataSourceContext, whose dataSources object contains
  listingAPI.
- src/resolvers.ts maps Query, Mutation, and Listing.amenities fields to
  functions. Query resolvers call the data source. Listing.amenities may make a
  follow-up REST call. Mutation.createListing returns a structured payload.
- src/datasources/listing-api.ts extends RESTDataSource and translates domain
  methods into GET and POST calls to an external listings REST API.
- codegen.ts generates src/types.ts from the schema with the typescript and
  typescript-resolvers plugins. Resolver context is typed as DataSourceContext.
- src/helpers.ts checks whether returned amenities appear to contain full
  objects.
- TypeScript compiles src to dist. The current build does not copy
  schema.graphql into dist, so the compiled server needs that build issue fixed
  before it can start successfully.

Teach interactively:
1. Begin by asking what part I want to study and what I already understand.
2. Explain one concept at a time in plain language, then connect it to a file
   and a concrete request flow in this project.
3. Ask me a short prediction or comprehension question before giving the
   answer.
4. When showing code, explain the role of each important line and distinguish
   required framework code from project-specific design choices.
5. Correct misconceptions directly but constructively.
6. Prefer small exercises that build on this server. Do not provide the full
   solution until I attempt the exercise or explicitly ask for it.
7. Point out tradeoffs, error cases, type-safety concerns, and production
   considerations when relevant.

My first topic/question is:
[REPLACE THIS WITH WHAT YOU WANT TO LEARN]
```

## 10. Short glossary

| Term | Meaning in this project |
| --- | --- |
| Schema / SDL | The GraphQL types and operations in `schema.graphql` |
| Type definitions | The parsed schema passed to Apollo as `typeDefs` |
| Resolver | A function that supplies the value of a GraphQL field |
| Root resolver | A resolver under `Query` or `Mutation` |
| Field resolver | A resolver for a nested field such as `Listing.amenities` |
| Context | Per-operation dependencies available to every resolver |
| Data source | A class that encapsulates access to the REST backend |
| Selection set | The fields inside braces that a client requests |
| Code generation | Creating TypeScript types from the GraphQL schema |
| Nullability | Whether a schema value may be `null`, controlled with `!` |
| N+1 problem | One initial request followed by many per-item requests |

The central mental model is: **schema defines the promise, resolvers fulfill
the promise, context provides the tools, and data sources communicate with the
underlying systems.**
