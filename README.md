# box-go-sdk

Official-style **Go client SDK** for the [MACROX BOX](https://box.macrox.pro) Public HTTP API.

Store and retrieve arbitrary JSON in Boxes and objects, list with Directus-style filters, manage external links, and handle structured API errors — in idiomatic Go.

> Status: **repository scaffolding**. Public client APIs are not implemented yet. Design: [CONCEPTION.md](CONCEPTION.md). Go practices: [docs/go-practices.md](docs/go-practices.md). Agent guide: [AGENTS.md](AGENTS.md).

## Module

```text
github.com/macrox-pro/box-go-sdk
```

Package name: `box`. Import path (module root — no nested `box/` directory):

```go
import "github.com/macrox-pro/box-go-sdk"
```

## Contract

HTTP behavior is defined by [https://box.macrox.pro/v1/_/openapi.yaml](https://box.macrox.pro/v1/_/openapi.yaml). That spec is not stored in this repository.

## Planned surface

```go
client, err := box.NewClient(
    box.WithBaseURL("http://127.0.0.1:8080"), // overrides the default https://box.macrox.pro
    box.WithAccessKey(os.Getenv("BOX_ACCESS_KEY")),
)
// client.Boxes()
// client.Objects()
// client.ExternalLinks()
```

See [CONCEPTION.md](CONCEPTION.md) for construction options, resource model, errors, pagination, filters, and acceptance criteria.

## Development

```bash
go test ./...
go test ./... -race
go vet ./...
```

Requires Go **1.25.7+**.

## License

[MIT](LICENSE) — Copyright (c) 2026 MACROX
