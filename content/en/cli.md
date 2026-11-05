---
title: "Command-line reference"
weight: 50
description: "The redfish command: sessions, raw requests, systems, and a mock BMC."
---

The `redfish` command ships with the module since **v0.3.0**. It is a [cobra](https://github.com/spf13/cobra)
command tree with POSIX flags and `--help` on every command. Results go to standard output and errors to
standard error. This page follows the `--help` of v0.8.0.

```sh
go install github.com/go-redfish/redfish/cmd/redfish@v0.8.0
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
| `redfish session list` | lists the active sessions of the service (v0.7.0) |
| `redfish session clear` | ends every active session of the service, its own last (v0.7.0) |
| `redfish session config` | changes `SessionService` properties (v0.7.0) |

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
| `systems logs` | the entries of the system's log services, and `logs clear` (v0.6.0) |
| `systems virtual-media` | get, insert, eject (v0.6.0) |

Each command's flags are in its `--help`.

## chassis {#chassis}

Since **v0.5.0**, `redfish chassis` reads and drives the Chassis resources: the Chassis commands of the
Ansible modules `redfish_command` and `redfish_info`. `--id` names one chassis (the last segment of its
path). Like the Ansible modules, the reading commands (`inventory`, `fans`, `power`, `thermals`, `psu`,
`health`, `hpe`) cover every chassis when no `--id` is given. The other commands act on one chassis: the
only one, or the one `--id` names. Without `--id`, a service with several chassis gets an error that lists
them.

| Command | What it reads or does |
| --- | --- |
| `chassis list` | lists the chassis (their paths) |
| `chassis inventory` | the identity of the chassis (`GetChassisInventory`) |
| `chassis power` | `PowerControl` of the `Power` resources (`GetChassisPower`) |
| `chassis thermals` | temperatures of the `Thermal` resources (`GetChassisThermals`) |
| `chassis fans` | fans of the `Thermal` resources (`GetFanInventory`) |
| `chassis psu` | power supplies present (`GetPsuInventory`) |
| `chassis health` | the health of the chassis, its power supplies, fans and PCIe devices (`GetHealthReport`) |
| `chassis led` | the indicator LED: on, off, blink |
| `chassis sensors` | the `Sensor` resources (`Sensors` collection) |
| `chassis power-subsystem` | `PowerSubsystem` and its power supplies |
| `chassis thermal-subsystem` | `ThermalSubsystem`, its fans and `ThermalMetrics` |
| `chassis metrics` | `EnvironmentMetrics`: power, energy, temperature, fan speeds |
| `chassis hpe` | HPE OEM settings covered by the Ansible modules: `thermal-config`, `fan-percent-min` |

The `Power` and `Thermal` resources are deprecated in favour of `PowerSubsystem`, `ThermalSubsystem`
and `Sensors`, but many BMCs still serve only them, so the command reads both.

## managers {#managers}

Since **v0.6.0**, `redfish managers` drives the Manager resources (the BMC itself): the Manager commands
of the three Ansible modules. `inventory`, `health` and `nic get` cover every manager when no `--id` is
given. The other commands act on one manager: the only one, or the one `--id` names.

| Command | What it reads or does |
| --- | --- |
| `managers list` | lists the managers (their paths) |
| `managers inventory` | identity and firmware (`GetManagerInventory`) |
| `managers health` | the health of the managers |
| `managers power` | `PowerGracefulRestart` (alias `GracefulRestart`), `PowerOn`, `PowerForceOff`, `PowerForceRestart`, `PowerGracefulShutdown`, `PowerReboot`; `--wait` blocks until the service answers again (`--wait-delay`, `--wait-timeout`) |
| `managers reset-to-defaults` | `ResetAll`, `PreserveNetworkAndUsers`, `PreserveNetwork` |
| `managers network-protocols` | the network services (HTTP, SSH, IPMI…): get, set |
| `managers ntp set` | the NTP service |
| `managers datetime` | the date, time and UTC offset: get, set |
| `managers nic` | the manager's network interfaces: get, set |
| `managers host-interface` | the host interfaces: get, set |
| `managers service-id` | `ServiceIdentification`: get, set |
| `managers logs` | the entries of the log services, and `logs clear` |
| `managers virtual-media` | get, insert, eject |

`managers virtual-media insert --image URL` puts the image in the first free slot that supports one of
the `--media-type` values. It uses the `InsertMedia` action when the slot has one, else a PATCH. Nothing
is sent when the image is already inserted. The password to fetch the image is read from
`--image-password-file` (mode 0600), never given on the command line.

v0.6.0 also added `systems virtual-media` (get, insert, eject) for services that attach virtual media to
the system, `systems logs` (entries and clear) and `chassis logs`.

## accounts {#accounts}

Since **v0.7.0**, `redfish accounts` drives the `AccountService`: the Accounts commands of
`redfish_command` and the `ListUsers` and `GetAccountServiceConfig` commands of `redfish_info`. An
account is named by its user name, or by `--id` (its Id).

| Command | What it does |
| --- | --- |
| `accounts list` | the accounts, without the empty slots (`ListUsers`) |
| `accounts add USERNAME` | creates the account; nothing is done when the user exists (`AddUser`) |
| `accounts delete USERNAME` | deletes the account; nothing is done when it does not exist (`DeleteUser`) |
| `accounts enable`, `disable` | `EnableUser`, `DisableUser` |
| `accounts password` | sets the account's password (`UpdateUserPassword`) |
| `accounts rename` | `UpdateUserName` |
| `accounts role` | sets `RoleId` (`UpdateUserRole`) |
| `accounts types` | sets `AccountTypes` and/or `OEMAccountTypes` |
| `accounts roles` | the roles of the service |
| `accounts config [set]` | shows the `AccountService` (`GetAccountServiceConfig`) or changes it |

A new password comes only from `--new-password-file` (mode 0600) or `--new-password-stdin`. No flag
takes it and no environment variable is read. It is never printed, and an error from a service that
echoes it back is redacted. Some services have a fixed number of account slots and refuse POST and
DELETE. On those, like the Ansible modules, `add` fills an empty slot (or the one `--id` names) with a
PATCH and `delete` empties the account.

## update {#update}

Since **v0.8.0**, `redfish update` drives the `UpdateService`: the Update commands of `redfish_command`
and `redfish_info`.

| Command | What it does |
| --- | --- |
| `update simple` | asks the BMC to fetch an image and update with it (`SimpleUpdate`) |
| `update push` | uploads an image to the `MultipartHttpPushUri`, streamed from the file (`MultipartHTTPPushUpdate`) |
| `update status` | the status of an update: its task or job (`GetUpdateStatus`) |
| `update operations` | performs what the update asks for to continue: its resets (`PerformRequestedOperations`) |
| `update inventory [--software]` | the firmware inventory, or the software inventory |
| `update capabilities` | what the service can do: multipart push, actions, transfer protocols |

`simple` and `push` start a task. Its monitor is printed on stderr and not awaited, like redfishtool's
`-N`. Add `--watch` to follow the task to its end; progress goes to stderr. `push` accepts only a regular
file and refuses one larger than `--max-size` (4 GiB by default) before it sends anything. The image is
streamed, never loaded whole into memory. The image server's password comes from
`--image-password-file` (mode 0600) or `--image-password-stdin`. A password written inside the image URI
is refused.

## Coming next {#next}

Commands for the last area (storage) come one minor release at a time, and each is marked in [Parity]({{< relref "parity.md" >}})
as it lands.
