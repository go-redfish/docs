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

What exists in v0.6.0:

| Capability | In v0.6.0 |
| --- | --- |
| service root and discovery | yes |
| Basic and session authentication | yes |
| energy counter and power reading | yes |
| energy from reads (`Meter`) | yes |
| generic requests (`Do`, `Get`, `Patch` with `If-Match`, `Post`, `Delete`, `WaitTask`) | yes |
| [mock BMC]({{< relref "mock-bmc.md" >}}) | yes |
| [`redfish` command]({{< relref "cli.md" >}}): sessions, raw requests, systems, mock | yes |
| Systems commands of the three modules | yes (v0.4.0; virtual media in v0.6.0), except storage (with Storage) |
| Chassis commands of the two modules | yes (v0.5.0) |
| Manager commands of the three modules | yes (v0.6.0) |
| accounts and sessions, update, storage | to come, one area per minor version |

At v0.6.0, 73 of the 99 commands are done. The 26 others are accounts and sessions (12), update (7) and
storage (7).

Any of the 99 commands can already be sent by hand with `redfish raw`. A dedicated command adds what the
Ansible module adds on top of the request: discovery of the right resource, a check of the allowed values
and waiting for the task.

## How parity is judged {#method}

The reference is the oracle: a behaviour is ported exactly, then compared with the reference's answer on
the same input, and a difference is either a defect to fix or a documented, deliberate departure. The
departures worth knowing already: the client refuses a non-https address and an HTTP redirect, on
purpose, whatever the reference accepts.
