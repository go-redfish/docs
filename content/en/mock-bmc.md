---
title: "Mock BMC"
weight: 40
description: "mockbmc: the DMTF public-rackmount1 mockup served as a stateful Redfish service."
---

## Why a mock {#why}

A test suite should never talk to a real machine: a BMC can power a server off, and nobody runs a suite
that needs one. The package [`mockbmc`](https://pkg.go.dev/github.com/go-redfish/redfish/mockbmc)
(since v0.3.0) is a Redfish service that a test starts in-process. A developer can also start it from the
command line with [`redfish mock serve`]({{< relref "cli.md#mock" >}}).

## What it serves {#serves}

The tree is the DMTF **public-rackmount1** mockup (DSP2043, BSD-3-Clause), vendored with its licence. It
has one system (`437XR1138R2`), one chassis (`1U`) and one manager (`BMC`), with the account, session,
event, update, task, certificate and key services, BIOS, secure boot, virtual media, log services, sensors,
`EnvironmentMetrics`, and both the older `Power`/`Thermal` resources and the newer
`PowerSubsystem`/`ThermalSubsystem`.

## Two trees {#trees}

A client tuned to one tree learns that tree's shortcuts, so since v0.4.0 the mock can serve a second one:
`mockbmc.Options{Tree: mockbmc.TreeEmulator}` serves `emulator-1u`, the static tree of the DMTF Redfish
Interface Emulator (BSD-3-Clause). That tree has no `Links.Sessions`, and its collections link their
members differently. The default is `mockbmc.TreeRackmount`. The library's Systems tests run against both
trees. `redfish mock serve` always serves the rackmount tree.

For BIOS settings, boot order and action checks, the mock adds what the static trees lack: a `BootOrder`,
`@Redfish.ActionInfo` resources, a BIOS attribute registry and BIOS `Settings` applied immediately.

## How it behaves {#behaviour}

Each rule comes from DSP0266 1.23.0:

- the service root and `/redfish` are open. Every other request needs HTTP Basic or an `X-Auth-Token`
  (13.3), else it gets 401 with a Redfish error;
- a POST to `SessionService/Sessions` logs in (13.3.4.2) and answers 201 with a `Location` naming the
  Session and an `X-Auth-Token` header. A DELETE of the Session logs out;
- every resource has an `ETag`. A PATCH whose `If-Match` does not match gets 412 (6.5, 7.6);
- a PATCH merges the properties sent into the resource and answers with the result;
- a POST to a collection creates a member (201 and `Location`) and a DELETE removes it;
- a POST to an action target is checked against the `AllowableValues` of its parameters.
  `ComputerSystem.Reset` changes `PowerState`;
- `UpdateService.SimpleUpdate` answers 202 with a task monitor in `Location` and a `Retry-After` header
  (12.2). The monitor answers 202 `TaskPolls` times, then 200 with the Task;
- an error is the Redfish error object with `@Message.ExtendedInfo` (8.6, 9.5.11).

State is kept in memory. `BMC.Log()` returns every request served (`METHOD path status`) and
`BMC.Resource(path)` returns a resource's current JSON, both for assertions.

```go
b, _ := mockbmc.New(mockbmc.Options{}) // admin / password
srv := httptest.NewTLSServer(b.Handler())
defer srv.Close()
// point redfish.Config.BaseURL at srv.URL and trust srv.Certificate()
```

## What it is not {#not}

It does not model any vendor's firmware. A client that passes against it follows the DMTF documents, which
does not prove that it works on your hardware. Faults such as a counter reset, a gap in power samples or
an oversized, malformed, slow or redirecting answer are not built into the mock. The client's own tests
inject them with dedicated `httptest` servers.
