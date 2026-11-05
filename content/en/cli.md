---
title: "Command-line reference"
weight: 50
description: "The redfish command: sessions, raw requests, systems, and a mock BMC."
---

The `redfish` command ships with the module since **v0.3.0**. It is a [cobra](https://github.com/spf13/cobra)
command tree with POSIX flags and `--help` on every command. Results go to standard output and errors to
standard error. This page follows the `--help` of v0.4.0.

```sh
go install github.com/go-redfish/redfish/cmd/redfish@v0.4.0
```

## Global flags {#global}

| Flag | Meaning |
| --- | --- |
| `--endpoint` | the BMC's address, `https://host[:port]` |
| `--user` | user name |
| `--password-file` | file holding the password (mode 0600) |
| `--session` | use the session saved by `redfish session login` instead of the password |
| `--ca-file` | PEM file of the CAs that sign the BMC's certificate (default: the system roots) |
| `--insecure` | do not verify the BMC's certificate; prints a warning on stderr and cannot be combined with `--ca-file` |
| `--timeout` | per-request timeout (default `10s`) |
| `-o`, `--output` | `json` (default) or `table` |

There is no password flag and no password environment variable. The password is read from a file, so it
never appears in a process listing or a shell history.

## session {#session}

| Command | What it does |
| --- | --- |
| `redfish session login` | logs in with `--user` and `--password-file` by POSTing to the Sessions collection (DSP0266 13.3.4). The `X-Auth-Token` is saved in a 0600 file under the user's config directory and is never printed |
| `redfish session status` | says whether a session is saved for `--endpoint` |
| `redfish session logout` | ends the saved session at the BMC and removes its file |

Afterwards, pass `--session` to any command instead of `--user` and `--password-file`.

## raw {#raw}

`redfish raw` sends one request to a path of the service (`/redfish/...`) and prints the body.

| Command | Request |
| --- | --- |
| `raw get PATH` | GET a resource |
| `raw head PATH` | HEAD: status and headers (ETag) |
| `raw patch PATH --data … [--if-match ETAG]` | PATCH properties; a stale `--if-match` is answered 412 |
| `raw put PATH --data …` | PUT: replace a resource |
| `raw post PATH --data …` | POST: create a member or invoke an action |
| `raw delete PATH` | DELETE a resource |

`--data` is the JSON body: the text itself, `@file`, or `-` for standard input. A 202 answer names a task
monitor: the monitor is printed on stderr and the Task body on stdout. A non-2xx answer is an error that
carries what the service said: its code, its message and `@Message.ExtendedInfo`.

## mock serve {#mock}

`redfish mock serve` starts the [mock BMC]({{< relref "mock-bmc.md" >}}) over HTTPS. It uses a self-signed
certificate valid for 24 hours, writes that certificate to `--ca-out` (pass it back as `--ca-file`), prints
the endpoint and stops on interrupt.

| Flag | Meaning |
| --- | --- |
| `--listen` | address to listen on (default `127.0.0.1:0`, a free loopback port) |
| `--ca-out` | where to write the certificate (default: under the user config directory) |
| `--mock-password-file` | file holding `admin`'s password; without it the password is `password` |
| `--task-polls` | how many times a task monitor answers 202 before the task completes (default 2) |

## Example {#example}

```sh
redfish mock serve --ca-out ./mock-ca.pem &        # prints https://127.0.0.1:PORT
printf 'password' > pw && chmod 600 pw
E=--endpoint=https://127.0.0.1:PORT
redfish $E --ca-file ./mock-ca.pem --user admin --password-file pw session login
redfish $E --ca-file ./mock-ca.pem --session raw get /redfish/v1/Systems
redfish $E --ca-file ./mock-ca.pem --session systems power PowerForceOff
redfish $E --ca-file ./mock-ca.pem --session session logout
```

## systems {#systems}

Since **v0.4.0**, `redfish systems` drives the ComputerSystem resources: the Systems commands of the
Ansible modules `redfish_command`, `redfish_config` and `redfish_info`. When the service has several
systems, `--id` names the one to use (the last segment of its path). Without `--id`, such a service gets
an error that lists the systems.

| Command | What it does |
| --- | --- |
| `systems list` | lists the systems (their paths) |
| `systems power` | `PowerOn`, `PowerForceOff`, `PowerForceRestart`, `PowerGracefulRestart`, `PowerGracefulShutdown`, `PowerReboot`, `PowerCycle`, `PowerFullPowerCycle`, each checked against the `ResetType` values the service allows |
| `systems boot-override` | the boot source override: get, set, disable |
| `systems boot-order` | the boot order: get, set, default |
| `systems bios` | BIOS attributes: get, set (through `@Redfish.Settings`), verify, reset, registry |
| `systems secure-boot` | UEFI Secure Boot: enable, disable |
| `systems power-restore-policy` | get, set |
| `systems led` | the system's indicator LED: on, off, blink |
| `systems asset-tag` | sets `AssetTag` |
| `systems inventory` | system, cpu, memory, nic, psu |
| `systems health` | the health of the system and its subsystems |

`redfish chassis led` sets the indicator LED of a chassis. Each command's flags are in its `--help`.

## Coming next {#next}

Commands for the other areas (chassis sensors and power, managers, accounts, update, storage, events,
certificates) come one minor release at a time, and each is marked in [Parity]({{< relref "parity.md" >}})
as it lands.
