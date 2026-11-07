# OAuth client (`a2aclient/oauth`)

> **Staleness warning**: if anything described here contradicts observed behavior, re-read
> the source code. This document may be out of date.

`oauth.Manager` gets, refreshes and stores OAuth grants for an A2A client. It hooks into a call in
two places, linked by context values (`interceptor.go`): the grant `Before` attached
(`withAttached` / `attachedFrom`), and a per-call `callState` that `Before` creates and the transport
fills with the result of following a 401's challenge, so `After` doesn't discover again.

## Call flow (JSON-RPC and REST)

```
Client call
  -> Interceptor.Before (interceptor.go)   card requires OAuth? attach grant or fail with InteractionRequiredError
    -> http.Client{Transport: Manager.Transport} (transport.go)
         attach a learned grant; on 401: refresh a rejected token, or follow the
         Bearer challenge to a stored grant; retry once; else return the 401 response
    <- transport returns *a2aclient.HTTPError for the 401
  <- Interceptor.After                       401 -> InteractionRequiredError{Err: the HTTPError}
```

- The transport returns an error only when it fails before sending (cleartext refusal, store
  failure) or when sending the request or its single retry fails. Every 401 it can't handle
  (refresh failed, discovery failed, no stored grant, body not replayable) is returned as is, and
  `After` decides what the caller sees. A request whose body can't be replayed gets no refresh or
  discovery in the transport.
- Redirect hops (`req.Response != nil`) get no learned grant and no 401 handling: net/http drops
  `Authorization` on redirects to unrelated hosts on purpose, and re-adding the target's grant
  would let agent A spend the user's grant for agent B.
- `After` uses the transport's discovery result from `callState`, else `Manager.RequirementForError`.
  `NoClientError`/`AmbiguousClientError` replace the 401 (`fmt.Errorf("%w: %w", typed, original)`,
  so the `HTTPError` stays reachable); other discovery failures keep the 401. Agent card fetches
  are not A2A calls: applications pass the `agentcard.ErrStatusNotOK` to `RequirementForError`
  themselves.
- `Before` marks calls that already carry an `Authorization` (`withCallerAuthorization`); `After`
  leaves their errors alone, as the transport leaves their requests alone. `After` blames the
  attached grant only when the call's final `ServiceParams` still carry it: a later interceptor
  (e.g. `AuthInterceptor`) may have replaced it.
- The transport refuses, before sending, a request that carries the token the Manager attached to a
  non-https, non-loopback URL, or to a URL the grant's resource doesn't cover. net/http copies
  `Authorization` on redirects to the same host or a subdomain, on any port and path and from https
  to http, and a REST call's URL (tenant, method path) is built after `Before` chose the grant.
  `resourceCovers` refuses paths with dot segments, which servers resolve but a prefix check doesn't.
- gRPC only gets `Before`: a gRPC error has no headers, so rejected tokens are not refreshed.

## Discovery limits and caches

- `fetcher.forAgent` refuses loopback URLs (metadata, redirects, authorization servers) unless the
  agent is on loopback, and with them authorization server endpoints on loopback
  (`checkEndpoints`), so a remote agent can't aim token or device POSTs at local ports. At most
  `maxAuthorizationServers` (5) servers from PRM are tried, and the metadata of at most
  `maxCardMetadataDocuments` (5) card schemes is fetched.
- A PRM document a challenge names (`resource_metadata`) must describe the request URL itself or a
  resource within `metadataScope` of its location (the path before `/.well-known/`, else its
  directory), so an agent on a shared host can't claim the whole origin. Well-known probes are
  host-controlled and accept any covering resource.
- An inline card flow has no issuer. `resolveInlineIssuer` adopts the metadata of the first
  registration whose `token_endpoint` is the card's; with `DefaultClientID` set and no match, it
  tries issuers at the token URL with path segments removed (`inlineIssuerCandidates`) and adopts
  one only if its metadata names the same token endpoint and supports CIMD. The issuer is never
  taken from the token URL alone.
- Metadata that lists `code_challenge_methods_supported` without `S256` loses its authorization
  endpoint (`usableEndpoints`), leaving the device flow, or is refused when it has none. A server
  that lists no methods is accepted.
- `RequirementForCard` caches per (card pointer, base URL) and resolved inline flows per declared
  `Endpoints`, both bounded (`boundedCache`); failures are cached for `failureCacheTTL`.
- `learned` (grants the transport attaches before a challenge) is bounded and pruned on `Delete`
  and `invalid_grant`.

## Grant resolution

`resolveGrant` (manager.go) is the only resolver. It is used by `lookup` (`Grant`),
`GrantForResource` and `learnedGrant`. Among the candidates whose resource covers the target
(`resourceCovers`: same origin, path prefix on a `/` boundary), it picks the longest resource. Two
candidates with the same resource are ambiguous, and it returns none. `lookup` first tries an exact
`Store.Get`, then filters `Store.List` by issuer, client, and the grant's effective scopes
(`Grant.scopes()`: granted if known, else requested); the exact hit must pass the same scope check.
`List` may omit tokens, so callers that need tokens `Get` the key. `Manager.get` treats a store's
`(nil, nil)` from `Get` as not found.

## Locking and refresh

- `Manager.lock`: the in-process semaphore first, then `Manager.Locker`, else the Store if it
  implements `Locker`. The wait ends with the caller's ctx or after `lockTimeout` (50 s), which is
  longer than a holder may keep the lock (`detachedTimeout` 30 s plus `storeTimeout` 10 s), so
  waiters don't fail while a slow refresh is still succeeding. The ctx given to `Locker.Lock` is canceled once `lock` returns.
- `refresh` re-reads the grant under the lock, so concurrent callers spend a rotating refresh token
  once. It checks the caller's ctx after taking the lock and starts no refresh for a caller that
  gave up. The `Get`, refresh request and `Delete` under the lock run on
  `context.WithoutCancel(ctx)` with `detachedTimeout`; the `Put` of a refreshed grant gets its own
  `storeTimeout`, so a refresh answered just before `detachedTimeout` still stores a rotated
  refresh token. `save` after sign-in also waits for the lock
  detached, since the grant it stores already exists. A caller's cancel can't lose a rotated
  refresh token.
- `invalid_grant` on refresh deletes the grant. A rejected token without a refresh token is cleared
  (`forgetAccessToken`) so it isn't sent again.

## Tests

`a2aclient/oauth/oauthtest` provides an authorization server (`AuthServer`, with rotation and
invalidation switches) and a protected echo agent (`ProtectedAgent`: card variants, `BasePath`,
`ProtectCard`, `JSONRPC`). Transport tests use a fake `roundTripFunc` base when they need exact
control over responses (redirects, rejected tokens).
