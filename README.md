<h1 align="center">TextConvo SDK for Go</h1>

<p align="center"><strong>Official Go SDK for the TextConvo API.</strong></p>

<p align="center">
  <a href="https://textconvo.ai">Website</a> &nbsp;&middot;&nbsp;
  <a href="https://textconvo.ai/docs">Developer Docs</a> &nbsp;&middot;&nbsp;
  <a href="https://textconvo.ai/docs#lead-ingestion-api">API Reference</a> &nbsp;&middot;&nbsp;
  <a href="https://textconvo.ai/contact-us">Support</a>
</p>

<p align="center">
  <img alt="Status" src="https://img.shields.io/badge/status-in%20development-db6d28?style=flat-square">
  <a href="https://textconvo.ai/docs"><img alt="Docs" src="https://img.shields.io/badge/docs-textconvo.ai%2Fdocs-1f6feb?style=flat-square"></a>
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/badge/license-MIT-2ea043?style=flat-square"></a>
  <img alt="Go" src="https://img.shields.io/badge/go-1.21%2B-00ADD8?style=flat-square">
</p>

---

> **Not released yet.** This repository is the public home of the official Go SDK. It holds the intended surface and the roadmap; there is no importable package yet. That is deliberate.
>
> **Use today:** [textconvo-api-examples](https://github.com/textconvo/textconvo-api-examples/tree/main/examples/go) has a dependency-free Go client with context timeouts, retries, idempotency, and HMAC signing. Copy it into your project.
>
> Watch this repository to hear about the first release.

## Planned installation

```bash
go get github.com/textconvo/textconvo-go-sdk
```

## Planned usage

A design sketch, not a contract, until v1.

```go
package main

import (
	"context"
	"errors"
	"log"
	"os"
	"time"

	"github.com/textconvo/textconvo-go-sdk/textconvo"
)

func main() {
	client, err := textconvo.NewClient(
		textconvo.WithAPIKey(os.Getenv("TEXTCONVO_API_KEY")),
		textconvo.WithSourceKey(os.Getenv("TEXTCONVO_SOURCE_KEY")),
		textconvo.WithHMACSecret(os.Getenv("TEXTCONVO_HMAC_SECRET")), // optional
		textconvo.WithTimeout(10*time.Second),
	)
	if err != nil {
		log.Fatal(err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
	defer cancel()

	// Idempotency, retries, and backoff handled for you.
	accepted, err := client.Leads.Ingest(ctx, textconvo.Lead{
		Phone:     "+15035551234",
		FirstName: "Jane",
		LastName:  "Doe",
		CustomFields: map[string]any{"roof_age_years": "12"},
	})

	var rateLimited *textconvo.RateLimitError
	switch {
	case err == nil:
		log.Printf("queued: %s duplicate=%v", accepted.IngestionRequestID, accepted.Duplicate)
	case errors.As(err, &rateLimited):
		log.Printf("rate limited, retry after %s", rateLimited.RetryAfter)
	default:
		log.Fatalf("ingest failed: %v", err)
	}
}
```

```go
// Webhook verification as standard middleware
http.Handle("/webhooks/textconvo", textconvo.VerifyWebhook(
	os.Getenv("TEXTCONVO_WEBHOOK_SECRET"),
	http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		event := textconvo.EventFromContext(r.Context())
		w.WriteHeader(http.StatusOK) // answer fast
		go process(event)            // work later
	}),
))
```

## Planned structure

```
textconvo-go-sdk/
├─ textconvo/
│  ├─ client.go           # NewClient, functional options, transport
│  ├─ errors.go           # typed errors, retryable vs terminal
│  ├─ leads.go            # LeadsService.Ingest
│  ├─ retry.go            # backoff with jitter
│  └─ webhooks.go         # VerifyWebhook middleware, event types
├─ internal/
├─ examples/
└─ go.mod
```

## Design commitments

**Standard library only.** No transitive dependencies. Go has everything this SDK needs.

**Contexts everywhere.** Every call takes a `context.Context` and honours cancellation.

**Errors that work with errors.As.** Typed errors rather than sentinel string matching.

**Safe by default.** Idempotency keys generated automatically, retries only for 429 and 5xx, exponential backoff with jitter, `hmac.Equal` for signatures.

**Functional options.** Additive configuration that never breaks compilation.

**Go 1.21+ and semantic import versioning.** v0 while the surface moves, v1 when it stops.

## Roadmap

| Milestone | Contents | Status |
| --- | --- | --- |
| v0.1.0 &mdash; Alpha | Client, options, `Leads.Ingest`, typed errors, retries, HMAC signing | Planned |
| v0.2.0 &mdash; Webhooks | `VerifyWebhook` middleware and typed event structs | Planned |
| v0.3.0 &mdash; Observability | `slog` hooks, request tracing, custom transports | Planned |
| v0.4.0 &mdash; Beta | Full test suite, godoc examples, chi and gin samples | Planned |
| v1.0.0 &mdash; GA | Stable API, compatibility promise | Planned |
| Post-v1 | Channel-send operations, contact retrieval, message status as those endpoints ship | Planned |

See [coverage](https://github.com/textconvo/textconvo-api-examples/blob/main/docs/COVERAGE.md) for what the API supports today.

## Feedback wanted, before the code exists

Functional options or a config struct? `any` or generics for custom fields? Middleware or a plain verify function for webhooks? [Open an issue](https://github.com/textconvo/textconvo-go-sdk/issues/new/choose) — opinions now are worth more than pull requests later.

## Contributing

Design feedback and documentation fixes welcome; implementation pull requests are on hold until the alpha surface is agreed. See [CONTRIBUTING.md](https://github.com/textconvo/.github/blob/main/CONTRIBUTING.md).

## Security

[SECURITY.md](https://github.com/textconvo/.github/blob/main/SECURITY.md) — never open a public issue for a vulnerability.

## License

[MIT](LICENSE) &copy; TextConvo
