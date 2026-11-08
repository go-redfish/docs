---
title: "Command-line reference"
weight: 50
description: "The redfish command: sessions, raw requests, systems, and a mock BMC."
---

The `redfish` command ships with the module since **v0.3.0**. It is a [cobra](https://github.com/spf13/cobra)
command tree with POSIX flags and `--help` on every command. Results go to standard output and errors to
standard error. This page follows the `--help` of v0.13.0.

```sh
go install github.com/go-redfish/redfish/cmd/redfish@v0.13.0
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
| `--no-expand` | read a collection's members one by one, never with `$expand` (v0.12.0; by default `$expand` is used when the service advertises it) |
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

### Query parameters {#queries}

Since **v0.12.0**, `raw get` can send the query parameters of DSP0266 1.23.0 section 7.3:

| Flag | Parameter |
| --- | --- |
| `--expand N`, `--expand-kind nolinks\|links\|all` | `$expand`, with N levels (7.3.2) |
| `--select A,B/C` | `$select` (7.3.3) |
| `--filter EXPR` | `$filter` on a collection's members, e.g. `"Status/State eq 'Enabled'"` (7.3.4) |
| `--top N`, `--skip N` | `$top`, `$skip` |
| `--only` | `only`: a collection of exactly one member answers with that member |
| `--excerpt` | `excerpt`: the properties of the schema's Excerpt annotation |

A parameter is sent only when the service root advertises it in `ProtocolFeaturesSupported`. Otherwise
the command fails and names the missing feature: a BMC that does not implement a parameter may ignore it
silently and return something other than what was asked.

With `$expand` on a collection, `$select` names member properties by their path under `Members`. For
example, one request returns every sensor's reading:

```sh
redfish $E --session raw get /redfish/v1/Chassis/1U/Sensors --expand 1 --select Members/Reading
```

`--select Reading` alone selects a property of the collection itself, which has none, so the answer
comes back without its members.

The commands that read a collection member by member now use `$expand` whenever the service advertises
it. Since v0.12.0 that covers inventories, sensors, power and thermal subsystems, firmware, accounts,
roles and subscriptions. v0.13.0 adds storage controllers, disks and volumes, host interfaces, virtual
media, log services and their entries, and the health reports. On the mock, `chassis sensors` drops from
43 requests to 2, and the system health report from 20 to 11, with output that is byte-for-byte the
same. `--no-expand` (global) goes back to member-by-member reads. They are also the fallback whenever an
expanded answer is refused, too large, or returns members as bare links.

Three reads stay member by member, by design:

- a storage's drives: a list of links inside the resource, not a collection;
- the virtual media slots that `insert` and `eject` act on: each slot's `Allow` header is needed, and an
  expanded member has none;
- the pages of log entries are never cut with `$top`, because a shorter page need not carry a link to
  the next one.

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

## storage {#storage}

Since **v0.9.0**, `redfish storage` drives the storage of a system: `GetStorageControllerInventory`,
`GetDiskInventory` and `GetVolumeInventory` of `redfish_info`, and `CreateVolume` and `DeleteVolumes` of
`redfish_config`. `--id` picks the system, as for `systems`.

| Command | What it reads or does |
| --- | --- |
| `storage controllers` | the storage controllers |
| `storage disks` | the drives of each controller, from `Storage`, or from the older `SimpleStorage` when the system has no `Storage` |
| `storage volumes` | the volumes of each controller |
| `storage volume create --storage ID` | creates a volume: `--raid-type`, `--drive` (repeatable), `--name`, `--capacity-bytes`; the RAID type and the drives are checked against the controller before anything is sent |
| `storage volume delete --storage ID VOLUME_ID...` | deletes the volumes named |

Deleting a volume destroys its data, so `volume delete` deletes only the volumes it is given by name.
It has no option to delete all of them. It prints the paths on stderr before the first deletion. If one
name is not a volume of the subsystem, nothing is deleted. `--dry-run` prints the paths and deletes
nothing.

With v0.9.0 every command of the three Ansible modules has its function and its command: see
[Parity]({{< relref "parity.md" >}}).

## events {#events}

Since **v0.10.0**, `redfish events` drives the `EventService`. This goes beyond the Ansible modules, and
the reference is the Redfish specification itself: DSP0266 1.23.0, section 12.1 (eventing) and section
12.5 (server-sent events).

| Command | What it does |
| --- | --- |
| `events status` | reads the EventService: whether it is enabled, its SSE URI, and the formats, registries, resource types and SSE filters it supports |
| `events subscriptions` | lists the subscriptions, or shows one |
| `events subscribe --destination URL` | creates a subscription, so the service POSTs events to that URL |
| `events unsubscribe` | deletes a subscription |
| `events test --message-id ID` | submits a test event (`SubmitTestEvent`) |
| `events listen` | opens the SSE stream and prints events as they arrive, until Ctrl-C or `--count` |

`subscribe` checks its filters (`--registry-prefix`, `--resource-type`, `--message-id`, `--origin`…)
against what the service supports before it sends anything. The destination must be an https URL with no
credentials, query or fragment. The service sends HTTP headers with every event, usually an
`Authorization` header for the receiver, and those headers are credentials. So they come from
`--headers-file` (mode 0600) or `--headers-stdin`, never from a flag, and are never printed.

`listen` prints one JSON line per event with `-o json`. `--filter` is passed as the stream's `$filter`
(for example `--filter "MessageId eq 'Contoso.1.0.TempAssert'"`), and the service applies it. The stream
is bounded: a line or an event larger than `--max-line` or `--max-event-bytes` (1 MiB each by default,
DSP0266 12.1.2) ends it with an error. After a broken connection, `listen` reconnects up to
`--reconnects` times, resuming with `Last-Event-ID`. An end by the service, a Ctrl-C or `--count` ends it
cleanly.

```sh
redfish $E --session events listen --count 1 &
redfish $E --session events test --message-id Base.1.0.Success
# {"event":{"@odata.type":"#Event.v1_6_0.Event","Events":[{"EventId":"1","MessageId":"Base.1.0.Success",…}],…},"id":"1"}
```

## certs {#certs}

Since **v0.11.0**, `redfish certs` drives the `CertificateService`. It follows DSP0266 1.23.0 sections
9.9.13 and 13.1.3, and the CertificateService 1.2.2, Certificate 1.11.0 and CertificateLocations 1.0.4
schemas.

| Command | What it does |
| --- | --- |
| `certs status` | reads the CertificateService: its locations resource and the actions it has |
| `certs list` | lists every certificate the service holds (`CertificateLocations`) |
| `certs show ID\|PATH` | shows what the service says about a certificate, and what the certificate itself says: subject, issuer, validity, key and usages, decoded with `crypto/x509` |
| `certs csr --collection PATH` | makes a certificate signing request (`GenerateCSR`) and writes the PEM to stdout or to a new file given with `--out` |
| `certs install` | adds a certificate to a collection (POST) |
| `certs replace` | puts a certificate in place of another (`ReplaceCertificate`) and prints the new one's path |
| `certs delete` | removes a certificate |
| `certs rekey`, `certs renew` | make a CSR from an existing certificate, with a new key pair or with the same one |

A certificate signed from one of the service's own CSRs needs no private key. For any other certificate,
the key comes from `--key-file` (mode 0600) or `--key-stdin`. It is checked against the certificate
before anything is sent, and it is never printed: an error that contains it is redacted. A certificate
string that already contains a private key is refused, so that the key can only reach the BMC through
that path. Passwords (the CSR challenge password, a PKCS#12 password) come from 0600 files, never from a
flag.

```sh
redfish $E --session -o table certs list
redfish $E --session certs csr --collection /redfish/v1/Managers/BMC/NetworkProtocol/HTTPS/Certificates \
  --common-name bmc.example.org --out bmc.csr
# csr: subject CN=bmc.example.org, ECDSA P-256, signature valid: true
```
