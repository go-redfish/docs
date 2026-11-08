---
title: "Security model"
weight: 30
description: "What the client refuses, what it bounds, and why a BMC's answer is untrusted input."
---

A BMC is a privileged network service, often old and rarely patched, and it holds the keys to the
machine. The client therefore treats what it says as data to check, and what it is given as secrets to
keep short-lived. Every rule below is in `github.com/go-redfish/redfish` at v0.15.0; the package
documentation cites the DMTF text it comes from.

## What the client refuses {#refuses}

| Refused | Because | Error |
| --- | --- | --- |
| a URL whose scheme is not `https` | a password travels with every request (Redfish Specification: Basic authentication requires HTTPS) | `ErrInsecure` |
| credentials in the URL | a URL ends up in logs, history and error messages | `ErrInsecure` |
| a query or a fragment in the base URL | the base is an address, not a request | `ErrInsecure` |
| a `MetricsPath` outside `/redfish/` | a configured path must stay inside the service | `ErrInsecure` |
| an HTTP redirect | a BMC that sends the client elsewhere would be sent the credentials there | the redirect response is returned as an error |
| turning certificate verification off | `Config` has no such setting; a caller that passes its own `Config.HTTP` client is responsible for that client's TLS settings | none |

Certificates are verified against the system roots or, with `Config.CAFile`, against the authorities in
that PEM file (a BMC usually has a private one). TLS 1.2 is the floor.

This is a difference from the tool it replaces. DMTF's `redfishtool` 1.1.8 never verifies a BMC's certificate:
every request it sends passes `verify=False` (`redfishtoolTransport.py`, lines 256, 392 and 640), so it sends
the password to whoever answers. Measured on the mock, verification costs `redfish` nothing visible (see
[Parity]({{< relref "parity.md#performance" >}})).

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

## The command {#command}

The `redfish` command follows the same rules, with these additions:

- there is no password flag and no password environment variable. The password is read from
  `--password-file`, whose mode should be 0600;
- `session login` saves the session token in a 0600 file under the user's config directory and never
  prints it. `session logout` ends the session at the BMC and removes the file;
- a password the command sends to the BMC (a new account's password, an image server's password) is
  read only from a 0600 file or standard input. In the library, an account password is a `Secret`,
  which prints as `[redacted]` in every format, and an error from a service that echoes it back is
  redacted (v0.7.0). The same holds for an image server's password since v0.8.0, and a password written
  inside an image URI is refused;
- `update push` streams the image from a regular file and refuses one larger than `--max-size` before
  it sends anything (v0.8.0);
- `storage volume delete` deletes only volumes named one by one. It prints them before the first
  deletion, deletes nothing if one name is unknown, and has `--dry-run` (v0.9.0);
- `events subscribe` accepts only an https destination without credentials, query or fragment. The HTTP
  headers the BMC will send with every event are read from a 0600 file or stdin, never printed, and
  redacted from errors. `events listen` refuses a line or an event larger than 1 MiB instead of
  buffering it (v0.10.0);
- a private key sent with a certificate (`certs install`/`replace`) comes only from a 0600 file or stdin,
  is checked against the certificate first, and is redacted from every error. A certificate string that
  already holds a key is refused, and a key in a BMC's answer is refused (CSR) or stripped (certificate)
  (v0.11.0);
- query parameters are sent only when the service advertises them, and are built by an encoder that
  refuses `&`, `#` and any name or value the grammar forbids, so a value cannot add a parameter (fuzzed).
  An expanded answer is bounded too, at 8 MiB instead of 1 MiB, and only for `$expand` requests
  (v0.12.0);
- `-o table` removes control characters from everything a BMC returns before it reaches the terminal:
  values, and since v0.12.1 property names too. Before v0.12.1 a property name such as
  `"\u001b[31mred"` reached the terminal as-is, so a malicious or compromised BMC could rewrite or hide
  what the operator sees. The fuzzer (`FuzzRenderTable`) found it, and the input that triggered it is
  kept as a regression seed. `-o json` was never affected, because JSON escapes control characters;
- the security commands (v0.15.0) only read. They never revoke a certificate, change a policy or create
  a key. Signed SPDM measurements are returned **without verifying their signature**, and the output says
  so. A key a service returns is dropped before output: only `key_present` remains;
- `--insecure` turns certificate verification off for one run. You have to ask for it explicitly, it
  prints a warning on stderr, and it is refused together with `--ca-file`.

## The mock BMC {#mock}

The [mock BMC]({{< relref "mock-bmc.md" >}}) is a test double. Its default password is `password`, it
keeps everything in memory, and `redfish mock serve` listens on loopback by default with a self-signed
certificate that is valid for 24 hours. Do not expose it on a network.
