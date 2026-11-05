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

What exists in v0.2.0, and nothing more:

| Capability | In v0.2.0 |
| --- | --- |
| service root and discovery | yes |
| Basic and session authentication | yes |
| energy counter and power reading | yes |
| energy from reads (`Meter`) | yes |
| information commands (`redfish_info` equivalents) | not released |
| action commands (`redfish_command` equivalents) | not released |
| configuration commands (`redfish_config` equivalents) | not released |
| mock BMC | not released |
| `redfish` command | not released |

## How parity is judged {#method}

The reference is the oracle: a behaviour is ported exactly, then compared with the reference's answer on
the same input, and a difference is either a defect to fix or a documented, deliberate departure. The
departures worth knowing already: the client refuses a non-https address and an HTTP redirect, on
purpose, whatever the reference accepts.
