# Go practices

Go form for this module. SDK behavior is [CONCEPTION.md](../CONCEPTION.md). Commands and the section index are [AGENTS.md](../AGENTS.md).

This file does not add API methods, retries, or options. If a Go pattern is not shown here, do not introduce it (no assert library, no test generator, no golden-file tree, no second HTTP client).

Package layout, `doc.go`, and the import path are CONCEPTION “Package structure”. Do not invent another layout here.

## Names

- Exported names are the compatibility surface. `MixedCaps`. Constructors `NewX`. Options `WithX`.
- Accessors and method names are CONCEPTION §5. Do not add `Foo` and `FooContext` pairs.
- Type parameters are CONCEPTION §5: `NewCollection[T any]`, `Collection[T]`, `Item[T]`, `RecordUpdate[T]`. `T` is the record struct. Do not add `ResourceClient[T, ID]`, `Do[T]`, or a constraint that merges string and numeric ids. `Boxes()`, `Objects()`, and `ExternalLinks()` stay non-generic.

```go
// Do
func (s *Objects) List(ctx context.Context, boxID string, opts ListOptions) ([]Object, error)

// Don't
func (s *Objects) List(boxID string, opts ListOptions) ([]Object, error)
func (s *Objects) ListContext(ctx context.Context, boxID string, opts ListOptions) ([]Object, error)
```

## Functions

- Do not add a helper that has one call site when that call site is still clear with the code inline.
- A large function may be split into smaller functions when the split is what makes the large function readable. A single call site is allowed in that case. The pieces are steps of that function, not a new API.
- Do not export a helper. If it is not part of CONCEPTION, it stays unexported in the file that uses it.
- Do not add a helper that only renames a stdlib or Resty call (`doGet`, `wrapErr` around one `fmt.Errorf`).

```go
// Don't — one call site, and the call site was already obvious.
func buildPageQuery(page *int) string {
    return strconv.Itoa(*page)
}

// Do — List is long; these names are the steps a reader needs.
func (s *Objects) List(ctx context.Context, boxID string, opts ListOptions) ([]Object, error) {
    if err := validateListOptions(opts); err != nil {
        return nil, err
    }
    // request, then decode
}

func validateListOptions(opts ListOptions) error { /* page and page_size rules */ }
```

## Options

Two different shapes. Do not mix them.

**Client construction** uses functional options. `Option` is not `func(*Client)`, and `NewClient` does not return `*Client` with no error.

- One constructor: `NewClient(opts ...Option) (*Client, error)`. Do not add `NewWithTimeout`, `NewWithTimeoutAndKey`, or a constructor per combination.
- `type Option func(*config) error`. `config` is unexported. An option does not take `*Client` and does not touch a Resty client. `NewClient` applies every option, then builds `Client` once.
- An option that cannot be applied returns an error. `NewClient` returns that error. It does not panic and it does not return a half-built client.
- A nil `Option` in the slice is an error. Do not call it.
- The same `WithX` may appear twice. The later call wins for that setting. Two different `WithX` functions both apply.
- Omitted options keep the defaults in CONCEPTION §2. Do not invent a default inside an option.
- Each exported option is `WithX`. Adding one does not change `NewClient`'s signature. The name and behavior are added to CONCEPTION in the same change.
- Do not export `config`. Callers pass `WithX(...)`, not a config struct.

```go
type config struct {
    baseURL string
    timeout time.Duration
}

type Option func(*config) error

func WithTimeout(d time.Duration) Option {
    return func(cfg *config) error {
        if d <= 0 {
            return fmt.Errorf("timeout must be positive")
        }
        cfg.timeout = d
        return nil
    }
}

func NewClient(opts ...Option) (*Client, error) {
    cfg := config{baseURL: "https://box.macrox.pro"}
    for _, opt := range opts {
        if opt == nil {
            return nil, fmt.Errorf("nil client option")
        }
        if err := opt(&cfg); err != nil {
            return nil, err
        }
    }
    return newClient(cfg)
}

// Don't
type Option func(*Client) // runs against a live client; cannot return error
func NewWithTimeout(baseURL string, d time.Duration) (*Client, error)
```

**One call** with several optional parameters uses a struct, not functional options. `ListOptions` is that struct. Fields that mean "omitted" are pointers or another presence bit already specified in CONCEPTION. Callers set fields by name. Do not use a positional composite literal in examples. Do not turn `Page` into `WithPage` on `List`.

```go
// Do
items, err := orders.List(ctx, ListOptions{Page: &p, PageSize: &n})

// Don't
items, err := orders.List(ctx, WithPage(2), WithPageSize(50))
```

## Errors

- Fallible functions return `error`.
- Strings are lowercase and have no trailing punctuation.
- Wrap with `%w`. Callers use `errors.Is` and `errors.As`. Do not drop the wrapped error.
- The message names the operation. It does not contain access keys, auth headers, or secrets.
- `APIError` fields, quota codes, and `204` are CONCEPTION §9. Do not store `context.Context` in the error.

```go
// Do
return fmt.Errorf("list objects: %w", err)

// Don't
return fmt.Errorf("List objects failed: %s", accessKey)
return errors.New("request failed") // status and extensions.code discarded
```

## Context

- `context.Context` is the first parameter of every function that performs network I/O (CONCEPTION §5).
- Do not store a `context.Context` in a struct.
- Pass the caller’s context through. Do not replace it with `context.Background()`.
- How that context is attached to Resty is CONCEPTION §3. Do not use `Client.SetContext` for per-call cancellation.
- Do not retry a canceled context or a client-side validation error (CONCEPTION §10).

## Concurrency

CONCEPTION §13. Additional Go rule: no mutable globals for clients, auth, or config. Per-call data stays on that call.

## Tests

Package `testing` only. Do not import testify, assert, require, go-cmp, or a golden-file helper. `go test ./...` must pass with no network and no live BOX. After a Go edit run `go test ./... -race` and `go vet` on the packages you changed.

The cases that must exist for SDK behavior are CONCEPTION §14. Do not add a case whose HTTP call is not in the OpenAPI spec. Do not drop a case §14 lists for behavior you implement. This section is how those cases are written.

A table-driven test is not a library. Each row is one complete case: inputs, the expected result, and a name. The test function is one loop. If you copy a test and edit a few lines, make those copies rows, or move the shared code into a helper that the rows call. If two cases do not share the same procedure, they are two `Test` functions, not one table with flags.

Write the loop body once and put the detail in the failure text. A failing row must show which case failed and what differed, without opening the test source. The subtest name is the input identity. The message is got and want. Do not print access keys, `Authorization`, or request bodies that contain secrets.

`t.Errorf` is not an assertion. The case keeps going after `t.Errorf`, so one row reports every broken field (method, path, query, body), not only the first. Use `t.Fatalf` only when the rest of that subtest cannot run: the server did not start, the body cannot be read, or the client is nil. Do not use `t.Fatal` for a value mismatch.

Compare scalars with `==`, bodies with `bytes.Equal`, and decoded structures with `reflect.DeepEqual`. Say which field failed.

### Container

Default: a slice of structs with a `name` field, in the order you want to read failures.

Use `map[string]struct{...}` only when the map key is the case name and no case depends on another case having already run. Map iteration order is not specified and is not stable between runs. A map case that fails only when it runs after another case is a broken test. Do not sort the map to hide that.

### Subtests and parallel

Call `t.Run(name, ...)` for every row. The name is `tt.name` for a slice, or the map key for a map.

`t.Parallel` on the parent test and on each subtest, both, only when that case does not mutate a shared client, a shared handler, a shared hit counter, or a shared `httptest.Server`. Parallel subtests run with each other only if the parent also called `t.Parallel`. This module is Go 1.25. The loop variable is per iteration. Do not write `tt := tt`.

Create the `httptest.Server` inside the subtest when the handler records the request or the cases run in parallel. A server created outside the loop and closed after `t.Run` without waiting will race. Call `t.Cleanup(srv.Close)` inside the subtest that created it.

### HTTP row

Each request row states the method, path, headers, query, and body it expects, and the status and body the fake server returns. The path is the operation you are implementing from the OpenAPI spec. Do not invent a path in the test that the spec does not have.

Check `X-Access-Key` equals the test key. Do not put that key in `t.Errorf` or in a log assertion by printing it. Assert it is absent from error strings instead.

A `204` row returns success and does not call `json.Unmarshal` on the body. A non-JSON error row still checks HTTP status and that the result is an `APIError` via `errors.As`, not a plain `invalid character` error with no status. A quota row checks `extensions.code` is exactly `OBJECT_QUOTA_EXCEEDED`, `BOX_QUOTA_EXCEEDED`, or `STORAGE_QUOTA_EXCEEDED`.

A write row (`POST`, `PUT`, `PATCH`, `DELETE`) whose server returns an error status, or whose listener closes the connection, counts handler hits. The want is 1. A count of 2 means a retry was added.

The skeleton below is the required order. The path and the method name come from the OpenAPI operation under test. Do not copy the path from this example. Compare each query key with `r.URL.Query().Get`. Do not compare `RawQuery` to one string: parameter order is not part of the contract. `Page` and `PageSize` are `*int` (CONCEPTION §8). Nil means the key is absent.

```go
func TestListObjectsQuery(t *testing.T) {
    page := 2
    size := 50
    tests := []struct {
        name     string
        page     *int
        pageSize *int
        status   int
        wantHits int
    }{
        {name: "page 2 size 50", page: &page, pageSize: &size, status: http.StatusOK, wantHits: 1},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            var hits int
            srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                hits++
                if r.Method != http.MethodGet {
                    t.Errorf("got method %s, want GET", r.Method)
                }
                q := r.URL.Query()
                if got, want := q.Get("page"), strconv.Itoa(*tt.page); got != want {
                    t.Errorf("got page %q, want %q", got, want)
                }
                if got, want := q.Get("page_size"), strconv.Itoa(*tt.pageSize); got != want {
                    t.Errorf("got page_size %q, want %q", got, want)
                }
                if r.Header.Get("X-Access-Key") != "test-key" {
                    t.Errorf("X-Access-Key differs from the key passed to WithAccessKey")
                }
                w.WriteHeader(tt.status)
            }))
            t.Cleanup(srv.Close)

            client, err := NewClient(WithBaseURL(srv.URL), WithAccessKey("test-key"))
            if err != nil {
                t.Fatalf("new client: %v", err)
            }
            t.Cleanup(func() { _ = client.Close() })

            // The test is the caller, so Background is the caller's context.
            // SDK methods must pass that context through. They must not swap in their own.
            // Arguments besides ctx and ListOptions come from the spec operation.
            _, err = client.Objects().List(t.Context(), "box-id", ListOptions{
                Page: tt.page, PageSize: tt.pageSize,
            })
            if err != nil {
                t.Errorf("got error %v, want nil", err)
            }
            if hits != tt.wantHits {
                t.Errorf("got hits %d, want %d", hits, tt.wantHits)
            }
        })
    }
}
```

`hits` is read after `List` returns, on the test goroutine. Do not read it from another subtest. Do not copy this function once per status code. Add rows. A write row sets `wantHits` to 1 even when `status` is 500. A row with a nil `page` or `pageSize` must assert that query key is absent. Do not dereference the nil pointer. Do not replace `t.Errorf` with a helper that calls `t.Fatal` on the first field.

`Example` functions live in `*_test.go` and must run under `go test`. A live BOX check is opt-in, behind a build tag or an environment variable, and is not part of `go test ./...`.

## Stability

- Semantic versioning. After v1, a break is a new major path (`/v2`).
- A break is a change to an exported signature or to error behavior callers match on (HTTP status, `extensions.code`).
- Do not export transport types. Callers must not change when the HTTP helper changes.
- Tag releases. Mark SDK-only helpers as SDK behavior. HTTP behavior is the OpenAPI spec.
