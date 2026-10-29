# MACROX BOX Go SDK — Conception

Source of truth for product goals, architecture, and design principles of the MACROX BOX Go client SDK.

Rules in this file are binding. HTTP calls, fields, statuses, and error codes that are not in [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml) do not exist. SDK behavior that is not in this file does not exist. Do not add either.

| Question | Document |
|----------|----------|
| Which HTTP call, status, field, and error code exist? | [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml) |
| How the SDK exposes that call | This file |
| Go names, error strings, context storage, test form, semver | [docs/go-practices.md](docs/go-practices.md) |
| Which other-SDK shapes are allowed | [docs/sdk-patterns.md](docs/sdk-patterns.md) — Decisions only. The evidence table is not a feature list |
| Which section to open, and which commands to run | [AGENTS.md](AGENTS.md) |

[AGENTS.md](AGENTS.md) is the index. It does not repeat the rules below. [docs/go-practices.md](docs/go-practices.md) does not add API behavior.

---

## Purpose

This repository ships an idiomatic Go client for the **MACROX BOX Public HTTP API**. Callers store and retrieve arbitrary JSON in Boxes and objects, manage external links, and handle structured API errors. This is not a generic database layer.

**In scope:** Public REST (`/v1/...`) that the OpenAPI spec defines now: Boxes, objects, filters/pagination, and external links, authenticated with `X-Access-Key`.

**Not in the spec yet:** a project-unique box alias, and schema management (create a collection box, read or change its schema). Do not add those methods, fields, or paths until the spec defines them. When it does, they are in scope. The `NewCollection[T](client, BoxRef)` signature stays; `BoxRef` gains the new address form. Schema calls are separate operations, not a type parameter on `NewCollection`.

**Out of scope:** Private gRPC admin APIs (users, storages, access-key issuance), ORM/query engines, generating a server schema from a Go struct, and any endpoint the spec does not define.

---

## Relationship to OpenAPI

| Role | Location |
|------|----------|
| Authoritative HTTP contract | [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml) |
| Production API origin | `https://box.macrox.pro` |

**Rules:**

- Paths, methods, status codes, query/header precedence, and error codes come from that spec. Do not copy the file into this repository.
- Do not invent endpoints, fields, query parameters, headers, or semantics that the spec does not define.
- SDK conveniences (filter builder, typed options, helpers) must be named as SDK behavior in docs. They must serialize to the JSON, query, and headers the spec requires.
- When the spec and this file disagree about HTTP, change the SDK to match the spec, then update this file. Do not keep a third behavior.

---

## API surface model

MACROX BOX hierarchy (storage-scoped via access key):

```text
Storage (via X-Access-Key)
└── Box (singleton record OR collection of objects)
    ├── record / objects
    └── external links
```

| Resource | Service accessor | Typical operations |
|----------|------------------|--------------------|
| Box | `client.Boxes()` | Create, Get, Replace/Update record, Delete, metadata |
| Object | `client.Objects()` | Create, Get, List, Update, batch Replace, Delete |
| External link | `client.ExternalLinks()` | Create, Get, List, Delete, find by resource |

**Singleton vs collection:** Chosen at Box create (`single`). Object APIs on a singleton Box are invalid (`400`). The SDK must keep singleton Box operations and collection object operations **explicit**. Never auto-detect the mode and switch methods. Never add a method that accepts a mode flag and picks the call.

**Auth:** Header `X-Access-Key`. Guest/unauthenticated read of open Boxes is allowed only where the spec marks security optional. Never put credentials in URLs or query parameters. Do not also send the key as `Authorization` unless the spec says to.

**Pagination:** `page` is 1-based, default 1. `page_size` is 1–1000, default 10. In Go, `Page` and `PageSize` are `*int`: nil means the caller omitted the field, so the SDK does not send that query parameter and the API default applies. A non-nil `0`, a negative page, or a size outside 1–1000 is an error returned before any HTTP call. Do not replace those values with 1 or 10. Object list responses are arrays without a total count; do not invent a `total` field on object lists. External-link lists may include `total` only when the spec says so. Do not switch the API to `limit` / `offset`.

**Filters:** Directus-style JSON over object `record` fields (nested paths, `_and` / `_or`, comparison and string operators, null checks). The wire parameter is the spec’s `FilterInQuery`. The builder must emit that JSON. Callers must still be able to pass raw filter JSON. Do not add operators the spec does not list.

**System fields:** `_key`, `_order` on object records have the spec’s semantics. Do not generate `_key` when the API assigns it. Do not treat a user field as a system field. Do not strip `_key` or `_order` from a retrieved record.

**Errors:** Body is a single `ErrorV1` (`message` + `extensions.code`). Quota codes, spelled exactly: `OBJECT_QUOTA_EXCEEDED`, `BOX_QUOTA_EXCEEDED`, `STORAGE_QUOTA_EXCEEDED`. Other codes are typically `HTTP_<status>` as the spec defines them. Do not rename these codes or map them to a different HTTP status than the response had. Deletes often return `204 No Content` with no body. Do not decode a body on `204`.

---

## Package structure

Module: `github.com/macrox-pro/box-go-sdk`

**This repository is the public package.** Module root = package `box`. Do not create a nested `box/` directory. A tree in a design note that shows `box/` as a folder is a file grouping, not a package path. Put those files at the repo root.

```text
# module root (package box)
client.go           # NewClient, Client, service accessors
options.go          # WithBaseURL, WithAccessKey, WithHTTPClient, WithTimeout, …
errors.go           # APIError, sentinels, helpers
models.go           # Public request/response types
boxes.go            # Boxes service
objects.go          # Objects service
filters.go          # Directus-style filter builder
external_links.go   # ExternalLinks service
collections.go      # NewCollection[T], Collection[T], Item[T]
internal/
    transport.go    # Resty wiring, shared request execution
    encoding.go     # JSON encode/decode helpers
```

Keep the package **flat** at module root. Add root `internal/` only to hide transport details or prevent unwanted exports. Do not add `doc.go` files. Naming, errors, context, and tests: [docs/go-practices.md](docs/go-practices.md). Do not add further packages (`api/`, `services/`, `client/`) unless this file is changed first.

Import path for callers:

```go
import "github.com/macrox-pro/box-go-sdk"
```

---

## Design principles

### 1. Core design goals

- **Idiomatic Go:** names, error strings, context storage, test form, and semver are [docs/go-practices.md](docs/go-practices.md). Follow that file for those topics. Do not import an assertion library or another HTTP client. Package layout is [Package structure](#package-structure) in this file.
- **API-first:** Model the REST API, not a generic database abstraction.
- **Explicit behavior:** No hidden network calls, implicit resource creation, silent transforms, or surprise retries. A method does the one HTTP call its name and the spec describe. It does not prefetch, follow up with another call, or create a missing parent.
- **Minimal dependencies:** HTTP transport is Resty v3, module `resty.dev/v3`, inside `internal/` only. Do not leak Resty into ordinary public signatures. Do not add a second HTTP library.
- **Extensibility:** New resources get their own service file and accessor when the spec adds them. Do not add a plugin, registry, or generic repository in advance.
- **Predictable compatibility:** Exported types, method signatures, and error behavior are a stability contract. Do not export a type “for convenience” if callers would then depend on it.

### 2. Client construction and configuration

- Single constructor: `NewClient(opts ...Option) (*Client, error)`. The Go shape is [docs/go-practices.md](docs/go-practices.md) Options: `Option` is `func(*config) error` on an unexported config, applied before `Client` exists. It is not `func(*Client)`. A bad option or a nil option returns an error. Do not panic. Do not add a constructor per combination (`NewWithTimeout`).
- Functional options include `WithBaseURL`, `WithAccessKey`, `WithHTTPClient`, and `WithTimeout`. Do not add an option that is not listed here or in a later section of this file without adding it to this file in the same change. If the same `WithX` is passed twice, the later one wins. A per-call value such as `Page` is a field on a struct (§8, §11), not a `WithPage` argument to `List`.
- Validate required configuration during construction. An invalid URL returns an error from `NewClient`. Do not panic.
- Production origin, used when `WithBaseURL` is omitted: `https://box.macrox.pro`. `WithBaseURL` replaces that origin. Do not default to localhost, an empty host, or any other host.
- Initialize resource services once in `NewClient`.
- Service fields are **unexported**. Expose them only through `Boxes()`, `Objects()`, and `ExternalLinks()`, so callers cannot replace service instances.
- Configuration is immutable after construction unless a setting is explicitly documented as mutable and concurrency-safe. None are, unless this file says so later.
- `WithHTTPClient` accepts `*http.Client` and is passed to `resty.NewWithClient`. Public options do not accept `*resty.Client` or any other Resty type.
- Do not export a mutable config map or struct.
- `Client` and its services are safe for concurrent use after `NewClient` returns, when custom middleware follows §13. State that in the `Client` doc comment.

```go
// Do — service value is not a field callers can assign.
type Client struct {
    objects objectsService
}

func (c *Client) Objects() *objectsService { return &c.objects }

// Don't
type Client struct {
    Objects *objectsService
}
```

### 3. Resty integration

Resty is the sole HTTP transport. Import `resty.dev/v3` ([resty.dev](https://resty.dev) is the v3 site; v3 is a release candidate, and the site currently documents `v3.0.0-rc.4`). Do not import `github.com/go-resty/resty/v2`. The v2 README is a different major. Keep every Resty type inside `internal/`.

- One `resty.Client` for the SDK client lifetime, created in `NewClient` with `resty.New` or, when `WithHTTPClient` is set, `resty.NewWithClient`. Call `R()` per operation. Do not build a new Resty client per request.
- Base URL: `Client.SetBaseURL`. v2 `SetHostURL` does not exist. Do not call it.
- `X-Access-Key`: `Client.SetHeader` once at construction. Do not use `SetAuthToken` or `SetHeaderAuthorizationKey`. Those write an auth-scheme header. The default scheme is `Bearer`, so `SetAuthToken` sends `Authorization: Bearer <token>`, not `X-Access-Key`.

```go
// Do
rc.SetHeader("X-Access-Key", key)

// Don't — this is Authorization: Bearer <key>, which this API does not use.
rc.SetAuthToken(key)
```
- Caller context goes on the request with `Request.SetContext` or `Request.WithContext` before `Get`, `Post`, or the other verb methods. `Client.SetContext` is copied onto each new request with `context.WithoutCancel`, so it does not carry cancellation or a deadline. Do not use it as the per-call context.
- Timeouts: `Client.SetTimeout`, overridden per call with `Request.SetTimeout`. Resty applies them with `context.WithTimeout`. It does not read `http.Client.Timeout`. If the request context already has a deadline, Resty leaves that deadline in place and does not apply `SetTimeout`. Set an explicit SDK timeout so a context with no deadline does not wait forever. Do not set `http.Client.Timeout` and assume Resty will honor it.
- Query: `Request.SetQueryParams` / `SetQueryParam` (the v2 field name `QueryParam` is `QueryParams` in v3). Resty encodes the query. Do not concatenate URLs or query strings for ordinary filters or pagination. Leave escaping on unless one call truly needs `SetQueryParamsUnescape`.
- Cross-cutting concerns (request IDs, logging, metrics) belong in request and response middleware, not in resource services. `AddRequestMiddleware` inserts before the default middleware. A non-nil error stops preparation. `SetRequestMiddlewares` replaces the chain; include `resty.MiddlewareRequestCreate` or `Request.RawRequest` is never built. `AddResponseMiddleware` appends after the defaults. `SetResponseMiddlewares` replaces that chain; keep `resty.MiddlewareResponseAutoParse` where the body must be read. Response middleware always runs; inspect `Response.CascadeError` before acting. v2 names `OnBeforeRequest`, `OnAfterResponse`, and `SetPreRequestHook` are absent. Do not call them. Retry side effects remain hooks: `AddRetryHooks` (`RetryHookFunc`). Do not use them to retry writes.
- The `error` returned by the execute method is transport failure, request preparation failure, or decoding failure. It is not “HTTP status >= 400”. HTTP status `>= 400` is `Response.IsStatusFailure` (v2 name `IsError`). Decode an error body with `SetResultError` (v2 name `SetError`). `Response.Body()` was removed; buffered bytes are `Bytes` or `String`, and those stay empty after automatic unmarshal unless `SetResponseBodyUnlimitedReads(true)`. `MiddlewareResponseAutoParse` returns immediately on `204`. Do not unmarshal a `204` body.
- JSON customization, if needed, is `AddContentTypeEncoder` / `AddContentTypeDecoder`. v2 `SetJSONMarshaler` / `SetJSONUnmarshaler` do not exist. Prefer encoding user records in the SDK when that keeps unknown fields intact.
- `Client.Close` runs close hooks and stops client-owned background work (certificate watchers, load balancers). It is safe to call more than once. The SDK owns the Resty client, so the public client exposes `Close` and calls the Resty `Close` once at shutdown. Do not close per request. Do not tell callers to close the client for any other reason.
- Resty clients are safe for concurrent use. `Client.Clone` is not safe for concurrent use. Do not use `Clone` to make a per-request client. Do not mutate client-level headers for one call; set those on the `Request`.
- Retry defaults (the SDK policy in §10 is stricter and forbids turning retries on for writes): count is `0` until `SetRetryCount`. Leave the count at `0`. Total attempts would be the first try plus the count. When a retry runs, the wait is 100ms–2s with capped exponential backoff and jitter, or a custom `SetRetryDelayStrategy` (v2 name `SetRetryAfter`). Default conditions are enabled (`SetRetryDefaultConditions`). They retry temporary URL and network errors, and they skip TLS verification failures, an invalid scheme, invalid headers, and too many redirects. HTTP 429 and 5xx are not default conditions. They are separate helpers — `RetryConditionStatusTooManyRequests` and `RetryConditionStatus5XX` — and must not be installed on the default client. Above a count of 0, Resty retries idempotent methods only. The v3 upgrade guide lists GET, HEAD, PUT, DELETE, OPTIONS, and TRACE; `v3.0.0-rc.4` also treats its QUERY method as idempotent. That list includes writes (`PUT`, `DELETE`). POST and PATCH retry only after `SetRetryAllowNonIdempotent(true)`. Do not call that. A canceled context ends the retry loop. `Retry-After` is read for 429 and 503 only while computing a wait for a retry that is already allowed. It does not turn retries on.

### 4. Authentication and security

- Support `X-Access-Key` only through the centralized request setup in §3. Do not set it inside each service method separately.
- Never log access keys, authorization headers, or sensitive payloads by default. A debug option, if added later, must still redact those values. Do not add that option until this file describes it.
- Credentials never go in URLs or query strings.
- A caller-supplied header must not replace `X-Access-Key` or another protected SDK header unless this file defines an explicit option that does so. No such option exists.
- Client-side checks are not an authorization boundary. The server enforces access rights. Do not hide a `403` or `401` behind a client-side “you cannot do this” that skips the request when the spec would send it.
- Unauthenticated operations (open Box read) must be separate, explicitly named methods where the spec allows missing security. Do not reuse an authenticated method and silently omit the key.
- Do not share mutable authentication state across independently constructed clients. Each `NewClient` has its own key and its own Resty client.

### 5. Resource-oriented API design

- Services: `Boxes()`, `Objects()`, `ExternalLinks()`.
- Method names: `Get`, `Create`, `Replace`, `Update`, `Delete`, `List`. Use a name only when it matches that HTTP operation in the spec. `Replace` is full replacement. `Update` is partial update. Do not use one of them for the other.
- Keep singleton Box APIs and collection object APIs on different methods. Do not auto-infer which one the caller meant.
- No method that combines unrelated operations behind a flag, a boolean mode, or an untyped `map[string]any` of options that mean different calls.
- Services stay thin. Shared HTTP lives in `internal/` on the central client. Services do not construct their own Resty clients.
- Do not add a generic repository that covers Boxes, objects, and external links, and do not add `Do[T any](ctx, method, path, body)`.
- `context.Context` is the first parameter of every network method. There is no second method with a `Context` suffix. The Go form is [docs/go-practices.md](docs/go-practices.md).
- `List` on `Objects` and on `Collection[T]` returns that page only. It does not return an iterator, a sequence the caller ranges to fetch more pages, or a paginator with `All`, `HasMorePages`, or `NextPage`. The next page is another call with the next `Page`. Why this shape lost in other clients: [docs/sdk-patterns.md](docs/sdk-patterns.md) Findings.

**Typed collection.** This is what generics are for. `NewCollection` binds one caller struct to the object operations of one collection Box. It does not replace `Boxes()`, `Objects()`, or `ExternalLinks()`. Those stay non-generic. `json.RawMessage` stays available on `Objects()`.

`NewCollection` does not call the network and does not check whether the Box is a singleton. Object calls on a singleton Box still return the API’s `400`. There is no `NewSingleton`.

```go
type BoxRef struct {
    ID string // today: path box_id, a sqids string, not an integer
}

func NewCollection[T any](c *Client, box BoxRef) (*Collection[T], error)
```

`BoxRef` is how a caller names a box. `ID` in examples is the form the spec has today (`box_id`). It is not the only form the type is allowed to grow. A later spec may add a project-unique alias. That becomes another field on this same struct. Do not add the field, a second constructor, or an alias path before the spec defines it. `NewCollection` keeps this signature either way.

Today a nil `*Client` or a `BoxRef` with an empty `ID` returns an error, because `ID` is the only spec-backed form. When the spec adds another form, exactly one form is set. Do not send two forms unless the spec says how they combine. Do not guess that an alias is the `box_id` path segment.

`T any`. No SDK interface. `T` must marshal to a JSON object. An array, string, number, or null is an error, and the request is not sent.

`Collection[T]` stores `*Client` and the `BoxRef`. It does not own an HTTP client. It does not store a schema. Today the path uses `BoxRef.ID`.

| Method | HTTP |
|--------|------|
| `List(ctx, ListOptions) ([]Item[T], error)` | `GET /v1/b/{box_id}/o` |
| `Get(ctx, key string) (Item[T], error)` | `GET /v1/b/{box_id}/o/{key}` |
| `Create(ctx, record T) (Item[T], error)` | `POST /v1/b/{box_id}/o` |
| `Replace(ctx, key string, record T) (Item[T], error)` | `PUT /v1/b/{box_id}/o/{key}` |
| `ReplaceMany(ctx, []RecordUpdate[T]) ([]Item[T], error)` | `PATCH /v1/b/{box_id}/o` |
| `Delete(ctx, key string) error` | `DELETE /v1/b/{box_id}/o/{key}` |
| `DeleteKeys(ctx, keys []string) error` | `DELETE /v1/b/{box_id}/o` with query `keys` |

`key` is the object key string (`format: sqids`). Do not parse it as a number.

```go
type Item[T any] struct {
    Key       string
    Size      uint64
    Order     int64
    Revision  uint64
    CreatedAt time.Time
    UpdatedAt *time.Time      // nil when updated_at is omitted
    Record    T               // decoded from the response field record
    Raw       json.RawMessage // exact record object
    External  json.RawMessage // response field external; nil when omitted
}

type RecordUpdate[T any] struct {
    Key    string
    Record T
}
```

Read: decode `record` into `Record` and keep those bytes in `Raw`. Metadata is not inside `T`. `external` is not inside `T`.

Write: `Create` and `Replace` send `json.Marshal(record)` as the body. That JSON object is the record. `ReplaceMany` sends one array. Each element is `{"_key": Key}` plus the fields of `Record`. `_key` is `RecordUpdate.Key`, not a field of `T`. One call is one request. It replaces the listed records. It does not delete objects left out of the slice and does not create missing keys.

`T` must not marshal a field named `_key` or `_order`. If it would, return an error and do not send.

A typed write sends `T` only. Keys that exist in `Raw` but not on `T` are not written. Use `Objects()` and `Raw` when those keys must be kept.

`T` is not a schema. `NewCollection` does not create a box and does not write or update a schema. Creating a collection and changing its schema are future spec operations. Until those operations exist, do not add them, and do not derive a schema by reflecting on `T`. When the spec adds them, they are separate methods. They do not become extra arguments of `NewCollection`.

```go
type Order struct {
    SKU string `json:"sku"`
}

orders, err := box.NewCollection[Order](client, box.BoxRef{ID: boxID})
created, err := orders.Create(ctx, Order{SKU: "a"})
got, err := orders.Get(ctx, created.Key)

// Don't — a mode flag, a resource name, or an id constraint.
_, _ = box.NewCollection[Order](client, box.BoxRef{ID: boxID}, true)
_, _ = box.NewResource[Order, int](client, "objects")
```

```go
// Do — the caller picks the operation. The SDK does not look up whether the box is single.
_ = client.Boxes().Get(ctx, boxID)
_, _ = client.Objects().List(ctx, boxID, ListOptions{})

// Don't
_, _ = client.Get(ctx, boxID, single bool)
```

### 6. JSON and data modeling

- Arbitrary JSON documents are a first-class feature.
- Use `json.RawMessage` for a record whose structure the SDK does not know.
- Callers may decode a record into their own struct. Provide a function that returns the raw record bytes or `json.RawMessage` so they can do that. Do not require them to use an SDK struct.
- Typed convenience methods are allowed only when they do not remove the raw-JSON path.
- API metadata and the user `record` payload are different Go fields. Do not merge them into one flat struct that can drop or overwrite user keys.
- Preserve unknown JSON fields. Do not unmarshal a record into a struct that has no `json:",inline"` / raw capture and then marshal it back. That drops keys. Do not use `json.Decoder` `DisallowUnknownFields` on user records.
- Do not wrap fields of `T` in a generic present/null/raw field type. Omitted versus zero for `Page` and `PageSize` is the `*int` rule in §8, and only there. Unknown record keys stay in `json.RawMessage` or `Item[T].Raw`. They do not move to a side object on the HTTP response.
- Do not force callers to define a Go struct or a schema before storing a document.
- For each public method, the doc comment states the accepted input: `json.RawMessage`, `map[string]any`, or a typed value. Do not accept `any` without stating how it is encoded.
- Do not encode and decode the same payload twice unless a documented step needs the second pass.

```go
// Do — metadata and the user record are different fields.
// JSON names and nesting come from the spec, not from this snippet.
type Object struct {
    Key    string          `json:"_key"`
    Record json.RawMessage `json:"record"`
}

// Don't — unmarshal into this and marshal back; every key except name is dropped.
type Object struct {
    Name string `json:"name"`
}
```

### 7. System fields and data integrity

- `_key` and `_order` are system fields. Their meaning is the spec’s meaning, not a guess.
- Do not require callers to generate `_key` when the API assigns it. Do not send an empty `_key` to force a server default unless the spec says that is how creation works.
- Object identity is an explicit parameter or an explicit field on the update type. Do not take identity only from a field buried inside an untyped map unless the spec’s body is that map and this file says so.
- Batch update maps each object key to the payload the spec defines (`_key` in the body when that is the spec). Do not send a different envelope (`{"objects":[...]}` or a map keyed by id) unless the spec uses that envelope.
- Do not reinterpret a user field whose name looks like a system field.
- For each operation, document whether `_key` and `_order` are accepted, generated, ignored, or rejected. If the spec does not say, do not pick one. State “as in the spec” only after you have read the operation, and quote the spec’s behavior in the method comment.
- Preserve atomic batch-update semantics. One SDK call is one HTTP request. Do not split a batch into per-item requests. Do not retry a subset after a failure.

### 8. Filtering and pagination

- Provide a typed, composable filter builder for the spec’s Directus-style filter.
- Support nested field paths, comparison operators, string operators, null checks, `_and`, and `_or`, and only the operators the spec lists. Do not add an operator because Directus has it.
- Serialize filters to the exact JSON object the spec’s `FilterInQuery` parameter requires.
- Send that JSON with `Request.SetQueryParams` / `SetQueryParam` (§3). Do not hand-build a URL-encoded query string for the common path.

```go
// Do — one query parameter. The name is the spec's name for FilterInQuery, not a name you pick.
req.SetQueryParam(filterParam, string(filterJSON))

// Don't
u := base + "?filter=" + url.QueryEscape(string(filterJSON)) + "&page=" + strconv.Itoa(page)
```

The parameter name is the name in the spec (`FilterInQuery` and `page` / `page_size`). Do not rename it.
- Raw filter JSON remains available for an operator or shape the builder does not cover yet. The raw value is sent unchanged. Do not parse and re-emit it in a way that drops keys.
- `Page` and `PageSize` are `*int` on the list options struct. Nil does not send that query parameter. The API then applies its own default (page 1, page size 10). That default is not a value the SDK writes into the request.
- A non-nil value is sent as given when it is valid: page ≥ 1, page size in 1–1000. A non-nil `0`, a negative page, or a size outside 1–1000 returns an error and does not send the request. Do not substitute 1 or 10.
- The server remains authoritative for values the client accepted. Client rejection is only for the invalid set above.
- `List` performs one HTTP call and returns that page. It does not follow `page` inside the SDK until the items run out.

```go
// Do — nil means omit. A set 0 is rejected before the call.
var opts ListOptions
opts.PageSize = ptr(0)
if opts.PageSize != nil && (*opts.PageSize < 1 || *opts.PageSize > 1000) {
    return fmt.Errorf("page size %d is invalid", *opts.PageSize)
}

// Don't — a bare int cannot tell "unset" from 0, and this hides the caller's value.
var pageSize int
if pageSize < 1 {
    pageSize = 10
}
```

### 9. Errors and response handling

- `APIError` contains the HTTP status code, the API `extensions.code`, the message, and structured details from the body when they exist. Callers use `errors.As` and `errors.Is`.
- A success call returns the resource (or `Item[T]`). It does not return a per-operation struct with one field per status (`JSON200`, `JSON401`) and it does not return `*http.Response` beside the value. Status and `extensions.code` live on `APIError` when the call failed.
- Preserve quota codes exactly: `OBJECT_QUOTA_EXCEEDED`, `BOX_QUOTA_EXCEEDED`, `STORAGE_QUOTA_EXCEEDED`. `errors.Is` may match those codes only if a sentinel is defined for that exact string. Do not invent extra sentinels for codes the spec does not list.
- Distinguish API errors from network failures, context cancellation, timeouts, and JSON decode errors. A transport error is not an `APIError`. Mapping of HTTP status uses `IsStatusFailure` and `SetResultError` (§3).
- `204 No Content`: return success and do not decode a body. Do not treat an empty body as a JSON error.

```go
// Do
if resp.StatusCode() == 204 {
    return nil
}

// Don't
if err := json.Unmarshal(resp.Bytes(), dest); err != nil {
    return err // 204 has no JSON body
}
```
- An error response might not be JSON. If it is not, still return an `APIError` with the HTTP status and a message that the body was not JSON. Do not drop the status. Do not fail only with `invalid character`.
- Do not collapse failures into `fmt.Errorf("request failed")` with no status and no code.
- Error strings name the operation (method and resource). They do not contain the access key, `Authorization`, or the raw sensitive payload.
- Wrap with `%w`. Error strings are lowercase and have no trailing punctuation. Do not store `context.Context` in `APIError`. Details: [docs/go-practices.md](docs/go-practices.md).

### 10. HTTP semantics and retry policy

- Use the HTTP method the spec assigns to that operation. Do not “upgrade” `PATCH` to `PUT` or send `POST` where the spec says `PUT`.
- **No automatic retries of writes.** `POST`, `PUT`, `PATCH`, and `DELETE` are each sent once. Leave Resty’s retry count at `0`. Do not call `SetRetryAllowNonIdempotent`. A count above 0 also retries `PUT` and `DELETE`, because Resty treats them as idempotent (§3). That is not allowed on the default client.
- Do not add idempotency keys, `Retry-After` handling that retries, or a second attempt on connection reset, `429`, or `5xx`. The spec does not define an SDK retry policy.
- A retry policy may be added only by changing this section first. It must be opt-in, must name status codes, backoff, and max attempts, and must still refuse writes unless the spec defines an idempotency mechanism for that exact operation.
- Do not retry context cancellation or an error returned before any HTTP call (client-side validation).
- Do not treat “HTTP says this method is idempotent” as permission to retry. Application semantics are the spec’s.
- Do not issue a follow-up GET, a compensating DELETE, or a per-item loop that changes replace, delete, or batch behavior.

```go
// Do — Resty retry count stays 0. One Delete call is one HTTP DELETE,
// including when the status is 500 or the connection fails.
rc.SetRetryCount(0)

// Don't
rc.SetRetryCount(2)
rc.SetRetryAllowNonIdempotent(true)
for _, item := range batch {
    client.Objects().Delete(ctx, item.Key) // spec batch is one request
}
```

### 11. Request options and headers

- An operation with several optional parameters takes one options struct. Do not use a long list of positional booleans or strings.
- Auth, content type, query, and headers are set in the shared transport, not reimplemented per method with different precedence.
- When the spec allows the same value in a header and a query parameter, follow the spec. Query wins when the spec says both are set. Do not invent the opposite precedence.
- Do not overwrite a value the caller set on the options struct.
- Request-specific headers exist only where the spec defines them (for example `X-External-Links`). A general `map[string]string` of arbitrary headers is not the primary API. Do not add one unless this file adds it.
- API option structs and Resty configuration stay in different types. Callers do not pass `resty.Request` settings through API options.

### 12. External links

- External links are their own service. Their lifecycle is not the Box or object lifecycle.
- Numeric resource ids and string resource ids stay distinct types or distinct fields, matching the spec. Do not parse a string id with `Atoi` or format a number into a string and send it where the spec expects the other kind.

```go
// Do — the spec's string id stays a string.
type StringResourceID string

// Don't
n, _ := strconv.Atoi(stringID) // "0012" and "12" become the same number; errors are discarded
```
- Alias resolution (`GET /v1/e` and the other find operations the spec defines) is documented on the method: what storage and box an alias selects. Do not add resolution steps the spec does not perform.
- A method that reads a public or guest resource must not be named or documented as ordinary authenticated object access.
- Creating links with `X-External-Links`, and the dedicated link endpoints, each follow the spec’s body and headers. Do not merge those two mechanisms into one helper that sometimes sets a header and sometimes calls another URL, unless the method doc says which spec operation it performs.

### 13. Concurrency and lifecycle

- The client is safe for concurrent calls after `NewClient`, provided middleware and the injected `*http.Client` are too. General rules: [docs/go-practices.md](docs/go-practices.md).
- Do not mutate shared client headers or client config inside one call. Per-call headers and context are set on that `Request` (`SetHeader`, `SetContext`).
- There are no mutable globals for clients, keys, or config.
- Document, on `WithHTTPClient` and on any middleware option, who owns the object and whether the SDK closes it. The SDK does not close a caller-supplied `*http.Client`.
- The public client provides `Close`. It closes the internal Resty client once, at process shutdown (§3). Do not close it per request. Callers do not need to call `Close` for correctness of requests. They call it to release Resty background work.

### 14. Testing strategy

Tests use package `testing` only. The form below is the whole rule. Do not replace it with testify, a golden-file tree, or a copied test per status code. The worked HTTP table is in [docs/go-practices.md](docs/go-practices.md). It is an example of this section, not a second policy.

- One procedure and many inputs: one table, one loop, `t.Run` per row. A row holds the inputs, the expected result, and a name. Two procedures that differ are two `Test` functions.
- Slice of structs unless order must not matter. Then a `map[string]struct` whose key is the name. Map order is unspecified. Cases in a map must not depend on each other.
- A mismatch uses `t.Errorf` with got and want. `t.Errorf` does not stop the row, so every broken field is reported. `t.Fatalf` only when that subtest cannot continue. The name identifies the case. Failure text does not contain secrets.
- `t.Parallel` on the parent and the subtest only when the case does not share a server, handler, counter, or client. Go 1.25: do not rebind the loop variable.
- The fake server is `httptest.Server`, created inside the subtest when it records the request or the row runs in parallel. Check method, path, headers, query, and body. The path is the spec operation under test. Compare each query key with `url.Values.Get`. Do not require one exact `RawQuery` string. Call the client, then read hit counts on the test goroutine. `Page` and `PageSize` nil versus non-nil is §8.

What the tests must prove:

- `go test ./...` does not use the network and does not need a live BOX. `go test ./... -race` is the race check. Live checks are opt-in (build tag or environment variable) and are not in the default run.
- Request construction and response decoding are tested with `httptest.Server`: method, path, headers, query, and body. The server is created inside the subtest when cases run in parallel or the handler records the request.
- Auth options, including a custom `*http.Client` passed through `WithHTTPClient`.
- Arbitrary JSON records, including unknown fields that must survive. Nested filters. Reserved `_key` and `_order`. Batch-update bodies in the spec’s shape.
- Empty `204` with no decode. Malformed JSON. Non-JSON error bodies. Unexpected content types. Network failures (the server closes the connection, or the client uses a transport that fails).
- `APIError` keeps the HTTP status and `extensions.code`, including the three quota codes.
- A failed or error-status `POST`, `PUT`, `PATCH`, and `DELETE` is sent once. The handler counts hits.
- Contract coverage is against the operations the SDK implements. Do not add a test that expects a call the spec does not define.

### 15. Documentation and versioning

- Document installation, `NewClient`, authentication, and one complete example for each of Boxes, objects, and external links.
- Examples cover both `json.RawMessage` and a caller-defined struct. Put them in `Example` functions in `*_test.go` so `go test` compiles and runs them. Do not leave the only copy in Markdown.
- Document pagination (1-based, size 1–1000, default 10), singleton vs collection, batch atomicity, and `APIError` matching.
- Module path: `github.com/macrox-pro/box-go-sdk`. What counts as a break, and the `/v2` rule after v1, are [docs/go-practices.md](docs/go-practices.md) Stability. Do not invent a second compatibility policy.
- HTTP behavior in docs matches the spec. SDK-only helpers are labeled as SDK behavior.
- When tracking a new public API revision, re-read [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml) and change the SDK to match. Do not keep a local copy of the spec in the repo.
- Transport module: `resty.dev/v3` (release candidate; live docs currently `v3.0.0-rc.4`). Not `github.com/go-resty/resty/v2`. The call rules are §3.

### 16. Suggested package structure

The tree in [Package structure](#package-structure) is the package. Do not nest a `box/` directory. Do not add files that are not in that tree unless the change also updates that tree in this file. `internal/` is transport and encoding only.

### 17. Recommended implementation order

Do the steps in this order. Do not start a later step by stubbing invented endpoints.

1. Client construction, options, and Resty integration (§2, §3).
2. Shared request execution and structured `APIError` (§9, §10).
3. Box metadata and singleton operations.
4. Collection object operations and arbitrary JSON records (§6, §7).
5. Pagination and the Directus-style filter builder (§8).
6. Batch object updates and atomicity tests (§7, §14).
7. External links (§12).
8. Examples and release docs (§15). Live integration tests stay opt-in.

---

## Acceptance criteria (initial release)

The SDK is ready for an initial release when all of the following are true:

- [ ] A caller can configure a client without naming a Resty type in application code.
- [ ] Resource services cannot be replaced through exported client fields.
- [ ] Arbitrary JSON round-trips without loss of unknown fields.
- [ ] Singleton and collection operations are different methods. The SDK does not choose between them.
- [ ] Filtering and pagination match the spec (`FilterInQuery`, `page`, `page_size`).
- [ ] API errors retain the HTTP status and `extensions.code`, including the three quota codes.
- [ ] HTTP `204` is success with no body decode.
- [ ] `POST`, `PUT`, `PATCH`, and `DELETE` are not retried.
- [ ] Public behavior is documented. Concurrency and `Close` are documented. The behaviors above are covered by `go test ./... -race` using table-driven `httptest` tests.

---

## Other SDKs

[docs/sdk-patterns.md](docs/sdk-patterns.md) records which foreign shapes are allowed. Its Decisions list points back at sections of this file. It does not add behavior. Its evidence table is not a feature list. Do not copy stripe, go-github, slack-go, godo, or a Directus client except where that Decisions list says Use.

---

## Non-goals

- Private gRPC or admin management clients (users, storages, access-key issuance).
- A generic multi-backend storage abstraction.
- Turning `T` into a server schema, or requiring codegen before a write.
- Schema or alias methods that the OpenAPI spec does not define yet.
- Auto-retrying writes.
- Exposing Resty, or any other HTTP library, as the public API.
- Endpoints, fields, error codes, or defaults that are not in the OpenAPI spec or in this file.
