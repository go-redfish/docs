---
title: "Security model"
weight: 30
description: "What the client refuses, what it bounds, and why a BMC's answer is untrusted input."
---

A BMC is a privileged network service, often old and rarely patched, and it holds the keys to the
machine. The client therefore treats what it says as data to check, and what it is given as secrets to
keep short-lived. Every rule below is in `github.com/go-redfish/redfish` at v0.2.0; the package
documentation cites the DMTF text it comes from.

## What the client refuses {#refuses}

| Refused | Because | Error |
| --- | --- | --- |
| a URL whose scheme is not `https` | a password travels with every request (Redfish Specification: Basic authentication requires HTTPS) | `ErrInsecure` |
| credentials in the URL | a URL ends up in logs, history and error messages | `ErrInsecure` |
| a query or a fragment in the base URL | the base is an address, not a request | `ErrInsecure` |
| a `MetricsPath` outside `/redfish/` | a configured path must stay inside the service | `ErrInsecure` |
| an HTTP redirect | a BMC that sends the client elsewhere would be sent the credentials there | the redirect response is returned as an error |
| turning certificate verification off | there is no such setting | none |

Certificates are verified against the system roots or, with `Config.CAFile`, against the authorities in
that PEM file (a BMC usually has a private one). TLS 1.2 is the floor.

## What the client bounds {#bounds}

- **Size.** An answer is read up to `MaxBody` (1 MiB); a larger one is `ErrTooLarge`, never an
  allocation of the size the server announces.
- **Time.** `Config.Timeout` bounds every request, and the contexts you pass cancel them.
- **Shape.** A reply that is not the JSON the schema describes is `ErrMalformed`. Readings that are
  negative, not finite or absurdly large are discarded by the decoders rather than passed on.

## Secrets {#secrets}

`Credentials` is asked for the user and the password when a request needs them, so the caller resolves
its secret reference at that moment and the library keeps nothing longer. Error messages are scrubbed of
the secrets they could carry. With `Config.Session`, the password is sent once, to the
`SessionService`; the token is then used, and `Close` logs the session out.

Keep the password out of command lines and environment variables, where every process of the machine can
read it: read it from a file whose mode you check.

## Fuzzing and CI {#fuzzing}

The two JSON decoders (`ParseEnvironmentMetrics`, `ParsePowerConsumed`) and the meter are fuzzed, and the
inputs that once found a defect are committed under `testdata/fuzz/`. CI runs `go vet`, `go test -race`, a
short fuzz run and `govulncheck` on every change.

## What it does not do {#limits}

- It does not pin a certificate by fingerprint, only by authority.
- It does not rate-limit: a caller that polls a BMC every second may exhaust the BMC, which is itself a
  small computer. A minute is a sensible interval for energy.
- The mock BMC and the command are not covered by this page until they are released.
