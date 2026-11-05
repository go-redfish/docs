---
title: "Command-line reference"
weight: 50
description: "The redfish command: a stub until it is released."
---

{{< callout type="warning" >}}
**Stub.** The command is being written in [`go-redfish/redfish`](https://github.com/go-redfish/redfish)
and is not in v0.2.0. Nothing below is a promise of a flag name; the reference will be written from the
command's own `--help` when it is tagged.
{{< /callout >}}

## Conventions it will follow {#conventions}

- A [cobra](https://github.com/spf13/cobra) command tree, POSIX flags, `--help` on every command, output
  on standard output and errors on standard error.
- The same safety rules as the library: an https address only, no password in the address, no flag that
  carries a password (it is read from a file), verification of the BMC's certificate against `--ca-file`.
- Commands grouped as the Ansible modules `community.general.redfish_info`, `redfish_command` and
  `redfish_config` group them: read, act, configure. The table of which is done is in
  [Parity]({{< relref "parity.md" >}}).

## Reference {#reference}

To come, with each command's flags, exit status and an example, generated from the command itself so that
the page cannot drift from the binary.
