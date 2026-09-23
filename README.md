mittwald mStudio for [`libdns`](https://github.com/libdns/libdns)
=======================

[![Go Reference](https://pkg.go.dev/badge/test.svg)](https://pkg.go.dev/github.com/libdns/mittwald)

This package implements the [libdns interfaces](https://github.com/libdns/libdns) for the DNS of [mittwald mStudio](https://studio.mittwald.de), allowing you to manage DNS records of domains in mStudio projects.

## Configuration

| Field | JSON | Description |
|---|---|---|
| `APIToken` | `api_token` | An mStudio API token of a user with access to the projects of the domains |

The provider finds a domain in the projects the token's user can access; no project ID is needed.

```go
p := &mittwald.Provider{APIToken: os.Getenv("MITTWALD_API_TOKEN")}
recs, err := p.AppendRecords(ctx, "example.com.", []libdns.Record{
	libdns.TXT{Name: "_acme-challenge", Text: "token"},
})
```

## Caveats

- **Records are stored per name.** The mStudio API keeps the records of each name in an object of its own, which it calls a zone (`example.com`, `www.example.com`, `_acme-challenge.example.com`). These are not DNS zones: DNS has one zone per domain with one SOA, and a name below it cannot be delegated. The provider creates the object of a name when a record is added to it and deletes it when its last record is removed; the domain's own, one with a set that mStudio manages and one with objects of names below it stay. A record for a name without an object takes longer until the nameservers serve it than a change for a name that has one (73 against 21 to 31 seconds, measured once in production), so allow ACME DNS challenges a propagation timeout of at least two minutes.
- **One set per type.** A name holds one set per type: A and AAAA together, CNAME, MX, TXT, SRV and CAA. All records of a set share one TTL: records written with a TTL give it to the whole set (to AAAA as well when A is written, and the other way round), as with `libdns/hetzner`. So `AppendRecords` that adds a record with another TTL changes the TTL of the records already in the set (a record that exists already is not added again, whatever its TTL), and `DeleteRecords` does not compare TTLs. A TTL of 0 keeps the set's TTL, or means mStudio's "auto" (served as 60 seconds) for a new set; other TTLs are raised to 60 seconds or lowered to one day.
- **Supported types:** A, AAAA, CNAME, MX, TXT, SRV and CAA. A CNAME is possible below the apex only; the API rejects one at the apex. Wildcard names (`*`) are not supported by mStudio.
- **Managed records.** mStudio sets the addresses of a name connected to an ingress and the mail exchangers of its mail service. `GetRecords` does not return them; writing A, AAAA or MX records to such a name replaces them.
- **Not atomic.** A call that fails may have applied part of its records.
- **Rate limit.** mStudio allows an API user a number of requests per period. When the limit is used up, requests wait for its reset, as long as the context allows.

## License

MIT
