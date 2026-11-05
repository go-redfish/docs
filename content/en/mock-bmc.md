---
title: "Mock BMC"
weight: 40
description: "A Redfish service for tests: status of the work and what it will serve."
---

{{< callout type="warning" >}}
**Not released.** The mock BMC is being written in [`go-redfish/redfish`](https://github.com/go-redfish/redfish)
and is not in v0.2.0. This page states what it is for; it will be replaced by its reference when it is
tagged. Until then, the client's own tests use a local `httptest` TLS server.
{{< /callout >}}

## Why a mock {#why}

Nothing in a test suite should talk to a real machine: a BMC can power a server off, and a suite that
needs one is a suite nobody runs. The mock BMC is a Redfish service in Go that a test starts in-process,
or that a developer runs from the command line, so that the client, the command and anything built on
them are tested against the protocol rather than against a vendor.

## What it is meant to serve {#serves}

- the service root, the `Chassis`, `Systems` and `SessionService` collections, with the links a real
  service gives (so that discovery is exercised);
- `EnvironmentMetrics` with an `EnergykWh` counter, and the older `Power` resource with its
  `PowerControl`, so that both paths of the `Meter` are tested;
- sessions (`X-Auth-Token`) and HTTP Basic authentication over TLS, with a certificate the test can
  trust through `CAFile`;
- faults on demand: a counter that resets, a power with a gap, an oversized answer, a malformed one, a
  slow one, a redirect: the cases a BMC produces and a unit test would not think of.

## What it is not {#not}

It is not a model of any vendor's firmware, and a client that passes against it has shown that it follows
the DMTF documents, not that it works on your hardware. Parity with real implementations is a separate
question, kept in [Parity]({{< relref "parity.md" >}}).
