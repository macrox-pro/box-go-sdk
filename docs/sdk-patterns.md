# SDK patterns

Notes from Go API clients and from issues where users said the API was hard to use. Read on 2026-10-30.

This file does not add endpoints, options, or error codes. If it disagrees with [CONCEPTION.md](../CONCEPTION.md), CONCEPTION wins and this file is wrong.

Three layers:

| Layer | What it is | What an agent may do with it |
|-------|------------|------------------------------|
| Decisions | The only rules in this file | Follow them. Each one names the CONCEPTION section that already requires it. |
| Findings | Why a shape hurt users, with the issue or discussion | Use them to refuse that shape. Do not copy the library's fix unless a Decision says Use. |
| Evidence | What another client actually does | Not a feature. A row here is not something to implement. |

There is no official Directus Go SDK.

## Decisions

1. **Use** `NewClient(opts ...Option) (*Client, error)` and `context.Context` as the first parameter of every network method. CONCEPTION §2 and §5. **Refuse** a second method with a `Context` suffix (slack-go `GetBotInfo` / `GetBotInfoContext`).
2. **Use** services stored unexported and returned by accessors (`Boxes`, `Objects`, `ExternalLinks`). CONCEPTION §2. **Refuse** exported fields a caller can replace (`stripe.Client.V1Customers`, `github.Client.Issues`, `godo.Client.Droplets`).
3. **Use** one `APIError` with the HTTP status and `extensions.code`, checked with `errors.As`. CONCEPTION §9. **Refuse** a different response struct per operation and per status (`JSON200`, `JSON401` on oapi-codegen clients) and a status invented from an error-code table (go-directus `Errors`).
4. **Use** one page per call, with caller-supplied `Page` and `PageSize` (`*int`, CONCEPTION §5 and §8). **Refuse** an iterator that walks every page inside `List` (stripe `Seq2` / `V1List.All`, go-github `*Iter`, AWS `HasMorePages` loops). The caller asks for the next page by passing the next `Page`.
5. **Refuse** automatic retries of writes. CONCEPTION §10. godo issue [#173](https://github.com/digitalocean/godo/issues/173) asked for retries of 503s. `NewFromToken` then retries 429 and 5xx, including writes. That fix is not BOX behavior.
6. **Use** an unexported HTTP stack. Public injection is `*http.Client` only. CONCEPTION §2 and §3. **Refuse** `Native() *resty.Client` and any method that takes `*resty.Request`.
7. **Use** one generic: `NewCollection[T any](c *Client, box BoxRef) (*Collection[T], error)`. `T` is the object record. CONCEPTION §5. **Refuse** a generic client for every resource (`ResourceClient[T, ID]`, `Do[T](ctx, method, path, body)`, `readItems(collection)`, a type parameter that is a collection name). `Boxes()`, `Objects()`, and `ExternalLinks()` stay non-generic. There is no `NewSingleton`.
8. **Use** `json.RawMessage` on the untyped object API, and `Item[T].Raw` on the typed collection, so unknown record keys are not discarded. CONCEPTION §5 and §6. **Refuse** a typed struct as the only form of a record. stripe keeps `LastResponse.RawJSON` because typed structs drop unknown fields. Do not add `LastResponse`. The raw bytes belong on the record (`Raw`), not on a side HTTP response.

`Page` and `PageSize` are `*int` so omitted is different from zero. That is the same problem Cloudflare solves with `Field[T]` and go-github solves with pointers ([#2563](https://github.com/google/go-github/issues/2563)). BOX uses `*int` only for those two query fields. Do not wrap every field of `T` in `Field[T]`, `F()`, or `Null[T]`. `T` is an ordinary struct. Zero versus omitted inside `T` is the caller's `json` tags and `omitempty`, not an SDK wrapper.

## Findings

Each finding is a user or maintainer statement. The BOX column is the Decision that already covers it.

| What users hit | Where | What they asked for | BOX |
|----------------|-------|---------------------|-----|
| `List` returned a generic iterator (`Seq2`). Page metadata (`HasMore`, raw body) was not available. Manual pagination needed a hack (request one extra item). | [stripe/stripe-go#2168](https://github.com/stripe/stripe-go/issues/2168), fixed in v85 by returning `*V1List[T]` and requiring `.All(ctx)` to iterate ([migration](https://github.com/stripe/stripe-go/wiki/Migration-guide-for-v85)) | One page's data and metadata, and iteration only when asked | Decision 4. `List` returns `[]Item[T]` for the requested page. No `.All`. |
| Pre-generics stripe iterators were confusing: `Next` both advanced and returned the value, unlike `bufio.Scanner` and `sql.Rows`. | [stripe/stripe-go#29](https://github.com/stripe/stripe-go/issues/29) | Loop condition advances; accessors do not | Decision 4. No custom iterator type. |
| Generated `*Iter` helpers looped forever when the endpoint did not use the pagination mode the generator assumed (`since` versus `page`, cursor versus offset). | [google/go-github#3976](https://github.com/google/go-github/issues/3976) | Pagination options that match the real endpoint | Decision 4. Only `page` and `page_size` from the BOX spec. No generated `Iter`. |
| Callers cannot tell "field absent" from "field zero" on request structs unless the field is a pointer. | [google/go-github#2563](https://github.com/google/go-github/issues/2563) | Pointers, or another presence bit, on optional request fields | Decision 8's note. `*int` for `Page` and `PageSize` only. |
| Adding `context.Context` by a second method (`Foo` and `FooContext`) doubled the API. Later releases still kept both while changing parameters. | [slack-go/slack#1034](https://github.com/slack-go/slack/pull/1034), [discussion #1289](https://github.com/slack-go/slack/discussions/1289) | Cancellation without a second name for every call | Decision 1. Context is the first parameter. No `Context` suffix. |
| 503s aborted deploys. The requested fix retries 429 and 5xx inside the SDK, and `NewFromToken` turns that on by default. Enabling retries also replaces the caller's `*http.Client`. | [digitalocean/godo#173](https://github.com/digitalocean/godo/issues/173), [`NewFromToken`](https://github.com/digitalocean/godo/blob/main/godo.go) | Optional retry for reads; they also retry writes | Decision 5. No retry of `POST` / `PUT` / `PATCH` / `DELETE`. `WithHTTPClient` is not replaced. |
| Generated clients return a different struct per operation. Status is `JSON200` / `JSON401` fields. The body was already read, so a shared error handler cannot see one response type. Callers repeat the status check. | [oapi-codegen#240](https://github.com/oapi-codegen/oapi-codegen/issues/240), [#2149](https://github.com/oapi-codegen/oapi-codegen/issues/2149) | One way to read status, body, and a structured error | Decision 3. One `APIError`. |
| A generic response type cannot stand in for every operation's success and error types. The maintainer called that the limit of an "easy" generated client. | [oapi-codegen#2160](https://github.com/oapi-codegen/oapi-codegen/issues/2160) | Shared validation across endpoints | Decision 7. Generics bind `T` to one collection's record, not to every endpoint. |
| AWS v1 paginators used one callback and one context for every page, which made errors and rate hard to handle. v2 uses `HasMorePages` and `NextPage(ctx)` per operation, not one generic page type. A hand-rolled generic iterator is a user sketch, not the SDK. | [aws-sdk-go-v2#439](https://github.com/aws/aws-sdk-go-v2/issues/439), [v2 GA post](https://aws.amazon.com/blogs/developer/aws-sdk-for-go-version-2-general-availability/) | Context per page, errors on the call that failed | Decision 4. One `List` call, one page, caller's context. No paginator type. |
| Cloudflare wraps every request field in generic `Field[T]` (`F`, `Null`, `Raw`) so zero, null, and omitted differ. Response structs are plain values and lose null versus missing unless the caller checks a side `JSON` meta object. | [cloudflare-go README](https://github.com/cloudflare/cloudflare-go/blob/e5815164/README.md) | Do not send a zero by accident | Do not adopt `Field[T]` for records. Presence for BOX list options is `*int` (CONCEPTION §8). Unknown record keys stay in `json.RawMessage`. |

## Generics

Allowed shape, already specified in CONCEPTION §5:

```go
orders, err := box.NewCollection[Order](client, box.BoxRef{ID: boxID})
created, err := orders.Create(ctx, Order{SKU: "a"})
p, n := 2, 50
items, err := orders.List(ctx, box.ListOptions{Page: &p, PageSize: &n})
```

`T` is the record object. `Item[T]` holds metadata (`Key`, `Size`, `Order`, `Revision`, timestamps) and `Raw`. `List` returns that page only.

Not allowed, even though other SDKs do it:

```go
for item, err := range coll.List(ctx, opts).All(ctx) { // stripe V1List[T]
}
for p.HasMorePages() { // AWS paginator
    p.NextPage(ctx)
}
type ResourceClient[T any, ID ~string | ~int] struct{} // one client, id kinds merged
func (c *Client) Do[T any](ctx context.Context, method, path string, body any) (T, error)
params.Name = cloudflare.F("x") // Field[T] on every field
```

`NewCollection` is not `readItems("orders")`. The box argument is a `BoxRef`. Today that value is `BoxRef{ID: boxID}` because the spec's path parameter is `box_id`. `ID` is the example of the current form, not a promise that a box can only be named by id. A project-unique alias is not a field until the spec adds it. The type argument is the record struct, not a collection name and not a schema.

## Evidence

Do not implement a row. The last column is the Decision that blocks copying it.

| Client | What it does | Decision |
|--------|----------------|----------|
| [stripe-go](https://github.com/stripe/stripe-go) | `stripe.Client` services are exported fields (`V1Customers`). `List` returns `*V1List[T]`. `.All(ctx)` auto-paginates. `.Data()` and `.Meta()` are the page. Default network retries and idempotency keys. `RawJSON` on the last response. | 2, 4, 5, 8 |
| [go-github](https://github.com/google/go-github) | `NewClient(opts) (*Client, error)`. Exported `Issues`. `List` returns `([]*T, *Response, error)`. `ListOptions` has `Page` and `PerPage`. Generated `*Iter` exists and has looped forever when the pagination mode was wrong ([#3976](https://github.com/google/go-github/issues/3976)). | 1, 2, 4 |
| [slack-go](https://github.com/slack-go/slack) | `OptionHTTPClient`. Retries off unless `OptionRetry`, which can resend a POST. `Foo` and `FooContext` both remain ([#1289](https://github.com/slack-go/slack/discussions/1289)). | 1, 5, 6 |
| [godo](https://github.com/digitalocean/godo) | `New(*http.Client, opts) (*Client, error)`. Exported service fields and exported `HTTPClient`. `NewFromToken` panics on error and retries 429 and 5xx four times. `List` returns `([]T, *Response, error)`. | 2, 4, 5 |
| [cloudflare-go v4](https://github.com/cloudflare/cloudflare-go) | Services on a client. Optional request fields are `param.Field[T]` via `F`, `Null`, `Raw`. Responses are plain values plus a `JSON` side struct for null versus missing. | Refuse `Field[T]` on `T`. `*int` only for page fields. |
| [aws-sdk-go-v2](https://github.com/aws/aws-sdk-go-v2) | One client per service. Functional options. Paginator is per operation (`HasMorePages`, `NextPage(ctx)`), not one generic list. | 4 |
| [oapi-codegen client](https://github.com/oapi-codegen/oapi-codegen) | `NewClient(server, opts) (*Client, error)`. Each call returns its own response struct and `*http.Response`. Errors are `JSON4xx` fields, not one error type ([#2149](https://github.com/oapi-codegen/oapi-codegen/issues/2149)). | 3, 7 |
| [Directus JS](https://directus.com/docs/guides/connect/sdk) | `request(readItems(collection, query))`. Filter object is data. List paging is `limit` / `offset` / `page`. | 7. Filter JSON is only `FilterInQuery` in the BOX spec. Do not replace `Page` / `PageSize`. |
| [go-directus-sdk](https://github.com/macrox-pro/go-directus-sdk) | `NewClient(baseURL, opts) (*Client, error)`. Resty v2, retry count 0, `*resty.Client` unexported. Commands `NewReadItems[T](collection).SendBy(client)`. Status from a code table. README shows reflection to reach Resty. Same org; not a BOX spec. | 3, 6, 7 |
| [altipla directus-go](https://github.com/altipla-consulting/directus-go/blob/main/client.go) | `NewClient` returns `*Client` with no error. Exported `ResourceClient[T, PK]`. `http.DefaultClient`. `DELETE` `204` is not decoded. | 2, 7. The `204` rule is CONCEPTION §9, not a reason to copy the client. |
| [go-common httpclient](https://github.com/thanhbvha/go-common/blob/main/httpclient/client.go) | `Native() *resty.Client`. `Execute` takes `*resty.Request`. Retry when `err != nil` or status is at least 500. | 5, 6 |
| [sentry-go](https://github.com/getsentry/sentry-go/blob/master/client.go) | Event capture, not a resource client. | Do not use it as a shape. |

Filter operators, only when the BOX parameter `filter` is unclear: [Directus filter rules](https://directus.com/docs/guides/connect/filter-rules). An operator that is not in the BOX spec is not allowed (CONCEPTION §8).

## Sources

- https://github.com/stripe/stripe-go/issues/2168
- https://github.com/stripe/stripe-go/issues/29
- https://github.com/stripe/stripe-go/wiki/Migration-guide-for-v85
- https://github.com/stripe/stripe-go/pull/2179
- https://github.com/google/go-github/issues/3976
- https://github.com/google/go-github/issues/2563
- https://github.com/slack-go/slack/pull/1034
- https://github.com/slack-go/slack/discussions/1289
- https://github.com/digitalocean/godo/issues/173
- https://github.com/digitalocean/godo/blob/main/godo.go
- https://github.com/aws/aws-sdk-go-v2/issues/439
- https://aws.amazon.com/blogs/developer/aws-sdk-for-go-version-2-general-availability/
- https://github.com/cloudflare/cloudflare-go/blob/e5815164/README.md
- https://github.com/oapi-codegen/oapi-codegen/issues/240
- https://github.com/oapi-codegen/oapi-codegen/issues/2149
- https://github.com/oapi-codegen/oapi-codegen/issues/2160
- https://directus.com/docs/guides/connect/sdk
- https://github.com/macrox-pro/go-directus-sdk
- https://github.com/altipla-consulting/directus-go/blob/main/client.go
- https://github.com/thanhbvha/go-common/blob/main/httpclient/client.go
