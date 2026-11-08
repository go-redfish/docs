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

## Cost of one command {#performance}

This was measured on 2026-11-06 against DMTF's `redfishtool` 1.1.8, the CLI that go-ansible's default Redfish
backend shells out to. Both tools ran against the same mock BMC on loopback over HTTPS, in three rounds.

| Wall clock, mean | `redfish` v0.9.0 | `redfishtool` 1.1.8 |
| --- | --- | --- |
| raw GET of the ComputerSystem (both print the whole resource) | 4.6–5.3 ms | 69–75 ms |
| "get the system" (discovery, then the resource) | 5.6–5.9 ms | 76–83 ms |

Most of the difference is the Python interpreter starting on every call: `redfishtool` uses 54–66 ms of CPU
time per run before and around its two requests. The comparison is per CLI invocation. It does not measure a
long-lived client making many requests or a real BMC, whose own latency would dominate both columns. The method,
every figure and the commands to reproduce them are in
[`BENCHMARKS.md`](https://github.com/go-redfish/redfish/blob/main/BENCHMARKS.md).

## Beyond the Ansible modules {#beyond}

The three Ansible modules stop at the commands above. From v0.10.0, go-redfish covers further areas of
the Redfish specification, which then serves as the reference, with section numbers from DSP0266
1.23.0 and the DSP8010 2025.4 schemas. `PARITY.md` lists them in its own section. Each row gives the
spec section, the function, the command and the test, and a test checks that section.

| Area | Since | Not covered (and why) |
| --- | --- | --- |
| EventService: subscriptions, test event, SSE stream (12.1, 12.5) | v0.10.0 | the receiving end of push events (that is a server's job, not a client's); SNMP, SMTP, Syslog, Kafka and OEM protocols; changing an existing subscription or the service |
| CertificateService: locations, show (x509), CSR, install, replace, delete, rekey, renew (9.9.13, 13.1.3) | v0.11.0 | decoding PKCS#7 and PKCS#12 (sent, not decoded: Go's standard library has no reader for them); automatic enrollment; SPDM and UEFI properties; changing a certificate in place; encrypted private keys (the schema has no passphrase parameter) |
| Query parameters: `$expand`, `$select`, `$filter`, `$top`/`$skip`, `only`, `excerpt` (7.3), each sent only when `ProtocolFeaturesSupported` advertises it | v0.12.0 | `$skiptoken` (opaque; the next link is already followed); `includeoriginofcondition` (belongs to the event stream, saves no request); OEM parameters. Since v0.13.0, `$expand` is used for every collection read member by member, except a storage's drives (links inside a resource), the virtual media slots that insert and eject act on (each needs its own `Allow` header), and `$top` on log entries (it could drop the link to the next page) |
| TelemetryService: metric definitions, report definitions (create, delete), metric reports as time series, triggers (read), test report, reports over the event stream (DSP8010 2025.4, DSP0266 12.5.2) | v0.14.0 | `ClearMetricReports`, `ClearTelemetryData`, `CollectTelemetryData` and the two Reset actions; writing triggers; wildcard expansion in metric properties; the SSE `MetricReportDefinition` filter |

## How parity is judged {#method}

The reference is the oracle: a behaviour is ported exactly, then compared with the reference's answer on
the same input, and a difference is either a defect to fix or a documented, deliberate departure. The
departures worth knowing already: the client refuses a non-https address and an HTTP redirect, on
purpose, whatever the reference accepts.
