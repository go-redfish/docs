---
title: "Parity"
weight: 60
description: "How far go-redfish matches the Python tooling it replaces."
---

## The target {#target}

The behaviour of the Ansible modules `community.general.redfish_info`, `redfish_command` and
`redfish_config`, which are themselves built on the Redfish Specification and the DMTF
[`redfishtool`](https://github.com/DMTF/Redfishtool). A Go program that needs a BMC's answers should not
have to shell out to a Python client to get them.

## Where it stands {#status}

The authoritative table is [`PARITY.md`](https://github.com/go-redfish/redfish/blob/main/PARITY.md) in the
`redfish` repository, committed with the code it describes, so that a command is marked done in the same
change that implements it. This page does not copy it, because a copy is what drifts.

What exists in v0.9.0:

| Capability | In v0.9.0 |
| --- | --- |
| service root and discovery | yes |
| Basic and session authentication | yes |
| energy counter and power reading | yes |
| energy from reads (`Meter`) | yes |
| generic requests (`Do`, `Get`, `Patch` with `If-Match`, `Post`, `Delete`, `WaitTask`) | yes |
| [mock BMC]({{< relref "mock-bmc.md" >}}) | yes |
| [`redfish` command]({{< relref "cli.md" >}}): sessions, raw requests, systems, chassis, managers, accounts, update, storage, mock | yes |
| Systems commands of the three modules | yes (v0.4.0; virtual media in v0.6.0; storage in v0.9.0) |
| Chassis commands of the two modules | yes (v0.5.0) |
| Manager commands of the three modules | yes (v0.6.0) |
| Accounts and Sessions commands | yes (v0.7.0) |
| Update commands | yes (v0.8.0) |
| Storage commands | yes (v0.9.0) |

**At v0.9.0, all 99 commands are done.** Each row of `PARITY.md` names the function, the CLI command and
the tests behind it. The repository's tests fail if a row is still marked to do, so parity cannot
silently regress. A dedicated command adds what the Ansible module adds on top of the request: discovery
of the right resource, a check of the allowed values, and waiting for the task.

Where go-redfish deliberately differs from the Ansible modules (stricter checks, an error where the
module would silently go on), the difference is commented in the code and listed in the repository's
README.

## How parity is judged {#method}

The reference is the oracle: a behaviour is ported exactly, then compared with the reference's answer on
the same input, and a difference is either a defect to fix or a documented, deliberate departure. The
departures worth knowing already: the client refuses a non-https address and an HTTP redirect, on
purpose, whatever the reference accepts.
