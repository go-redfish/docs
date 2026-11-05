---
title: "Overview"
weight: 10
---

# go-redfish documentation {#go-redfish-documentation}

**go-redfish** is a set of Go libraries for the [DMTF Redfish](https://www.dmtf.org/standards/redfish)
API of a server's baseboard management controller (BMC). Everything is **pure Go** (`CGO_ENABLED=0`,
standard library only) and BSD-3-Clause.

## What is here {#what-is-here}

| Page | What it says |
| --- | --- |
| [Getting started]({{< relref "getting-started.md" >}}) | install the `redfish` module, read a node's energy, turn reads into kWh with the `Meter` |
| [Security model]({{< relref "security-model.md" >}}) | what the client refuses, what it bounds, what a BMC's answer is trusted for (nothing) |
| [Mock BMC]({{< relref "mock-bmc.md" >}}) | a Redfish service for tests, so that nothing is tested against a real machine |
| [Command-line reference]({{< relref "cli.md" >}}) | the `redfish` command: sessions, raw requests, the mock BMC |
| [Parity]({{< relref "parity.md" >}}) | how far the client matches the Python tooling it replaces, command by command |

## Status {#status}

The organisation has one repository with code, [`redfish`](https://github.com/go-redfish/redfish), whose
**current release is v0.3.0**. It has the client (service discovery, Basic or session authentication, a
CA file, bounded answers, typed errors), generic requests (`Do`, `Get`, `Patch` with `If-Match`, `Post`,
`Delete`, `WaitTask`), the `Meter`, the [mock BMC]({{< relref "mock-bmc.md" >}}) and the `redfish`
command. Commands for each area of a BMC (systems, chassis, managers and so on) come in the next minor
releases, and [Parity]({{< relref "parity.md" >}}) says how far they have got.

## Links {#links}

- Source: [github.com/go-redfish](https://github.com/go-redfish)
- Landing page: [go-redfish.github.io](https://go-redfish.github.io/)
