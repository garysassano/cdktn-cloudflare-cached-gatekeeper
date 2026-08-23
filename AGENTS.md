# Agent notes

Map of the code, and the invariants that are easy to break by accident.

## Layout

| File | |
| --- | --- |
| `src/stacks/my-stack.ts` | The stack: Worker script, per-entrypoint cache, secrets, subdomain. |
| `src/worker/index.ts` | Gatekeeper: capability methods, authorization, HTTP surface. Not cached. |
| `src/worker/upstream.ts` | The cached entrypoint. Holds the credential. Unreachable from the network. |
| `src/worker/cache-key.ts` | Canonicalization. |
| `src/worker/policy.ts` | Scopes, glob matching, the scope digest. |
| `src/worker/resources.ts` | Resource catalogue: paths, freshness, tags, upstream allowlist. |
| `src/worker/github.ts` | The only module that reads `GITHUB_TOKEN`. |
| `src/worker/explorer.ts` | The page that makes the cache visible. |
| `scripts/bench.ts` | Measured hit rate and upstream quota spend. |

## Invariants

Each of these is load-bearing. Breaking one leaves the project still building and still passing tests, while silently losing the property it exists for.

1. **Only `github.ts` reads `GITHUB_TOKEN`.** The point of a gatekeeper is one auditable place where the credential is used. Reading it anywhere else defeats the design.
2. **The gateway entrypoint must stay uncached.** `cache: { enabled: false }` on `default` is what guarantees every call is authorized, including the ones the cache answers. Enabling it would serve responses without an authorization check.
3. **Caching only ever happens on the `Upstream` entrypoint,** reached via `ctx.exports`. Workers Cache does not cache RPC, so any new capability method must route through a loopback `fetch()` to be cacheable.
4. **The canonical cache key must remain a valid GitHub REST path.** `resources.ts` builds the key and the fetched URL from the same value. Deriving them separately lets them drift, and a drifted cache serves one resource under another's key.
5. **Purging runs inside `Upstream`, never the gateway.** `ctx.cache.purge()` acts on the calling entrypoint's cache, and the gateway has none, so purging there silently does nothing while reporting success.
6. **`ctx.props` carries a digest of the caller's read scope, not their identity.** Identity would be safe but would cache nothing. Widening what goes in there changes the security boundary.
7. **The loopback request must carry no `Authorization` or `Cookie` header.** Workers Cache bypasses requests that do, which would disable caching entirely.
8. **`wrangler.jsonc` and `src/stacks/my-stack.ts` must agree** on cache config and compatibility date. Wrangler drives `pnpm dev`, Terraform drives the deploy. A test asserts this; keep it passing.

## Working on it

```sh
pnpm check         # lint, both typechecks, tests, synth
pnpm dev           # local worker; note that Workers Cache does NOT apply locally
pnpm run deploy    # needs CLOUDFLARE_API_TOKEN
```

Verifying a cache change requires a deploy. `wrangler dev` runs the logic but never caches, so a local run cannot confirm a caching claim. Prove hits by watching `originId` stop changing, or run `pnpm bench` against the deployed URL.

No em dashes in prose, here or in the README.
