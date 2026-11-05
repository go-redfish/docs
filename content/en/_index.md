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
| [Command-line reference]({{< relref "cli.md" >}}) | the `redfish` command: a stub until the command is released |
| [Parity]({{< relref "parity.md" >}}) | how far the client matches the Python tooling it replaces, command by command |

## Status {#status}

The organisation has one repository with code, [`redfish`](https://github.com/go-redfish/redfish), whose
**current release is v0.2.0**: the client (service discovery, Basic or session authentication, a CA file,
bounded answers, typed errors) and the `Meter`. The mock BMC, the command and the parity table are the
work in progress of the next releases; each page says what exists today and what does not.

## Links {#links}

- Source: [github.com/go-redfish](https://github.com/go-redfish)
- Landing page: [go-redfish.github.io](https://go-redfish.github.io/)
