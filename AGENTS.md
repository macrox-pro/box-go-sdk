# AGENTS.md — MACROX BOX Go SDK

Module `github.com/macrox-pro/box-go-sdk`. Package `box` at the module root. Import `github.com/macrox-pro/box-go-sdk`. Go 1.25.7+.

This file is the index. It does not repeat design rules. Open the owning document and follow that text. If a behavior is not written there, do not invent it.

## Authority

The first row that answers the question wins. Do not combine a lower row into a third behavior.

| Question | Open this | Ignore for this question |
|----------|-----------|---------------------------|
| Does this HTTP call, field, status, or error code exist? | [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml) | Every other doc. Do not copy the spec into the repo. |
| How does the SDK expose that call? | [CONCEPTION.md](CONCEPTION.md) | `docs/sdk-patterns.md` |
| Go names, errors, context storage, tests, semver | [docs/go-practices.md](docs/go-practices.md) | CONCEPTION §14 names the cases; the test **form** is go-practices |
| Is this other-SDK shape allowed? | [docs/sdk-patterns.md](docs/sdk-patterns.md) Decisions only | The evidence table in that file |

If this file disagrees with the owner, the owner wins. Fix this file.

## Commands

```bash
go test ./... -race
go vet ./...
go test ./... -run TestName
golangci-lint run ./...   # when installed
```

`go test ./...` must not use the network. After a Go change, run race and vet on the packages you touched.

## Where the rules are

Do not restate these in code comments as a second policy. Section numbers are CONCEPTION headings. Function extraction and the options pattern are [docs/go-practices.md](docs/go-practices.md) Functions and Options, not CONCEPTION sections. CONCEPTION §2 names which `WithX` exist.

| Topic | Section |
|-------|---------|
| Scope and non-goals | Purpose, Non-goals |
| Client, options, default origin `https://box.macrox.pro` | §2 |
| Resty v3 (`resty.dev/v3`), headers, timeout, `204`, `Close` | §3 |
| `X-Access-Key`, logging, guest calls | §4 |
| `Boxes` / `Objects` / `ExternalLinks`, `NewCollection[T]` | §5 |
| `json.RawMessage`, unknown fields | §6 |
| `_key`, `_order`, batch = one request | §7 |
| Filters, `Page`, `PageSize` | §8 |
| `APIError`, quota codes | §9 |
| No write retries | §10 |
| Option structs, query vs header | §11 |
| External-link ids | §12 |
| Concurrency, `Close` | §13 |
| Which tests must exist | §14 |
| Examples, versioning | §15 |
| File tree | Package structure |
| Build order | §17 |

## File map

```text
client.go options.go errors.go models.go
boxes.go objects.go filters.go external_links.go collections.go
internal/transport.go encoding.go
```

No other directories unless Package structure in CONCEPTION is updated in the same change. No `doc.go`. No vendored OpenAPI.

## Stuck

1. The operation in the OpenAPI URL. Missing there means do not add it.
2. The CONCEPTION section in the table. Missing there means do not add it.
3. [docs/go-practices.md](docs/go-practices.md) only for Go form and tests.
4. [docs/sdk-patterns.md](docs/sdk-patterns.md) only to accept or reject a shape. A fact about another client is not a feature.
