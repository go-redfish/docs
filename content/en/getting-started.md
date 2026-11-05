---
title: "Getting started"
weight: 20
description: "Install the redfish module, read what a node consumes, and turn reads into energy."
---

## Install {#install}

```sh
go get github.com/go-redfish/redfish@v0.5.0
```

The library has no dependency outside the standard library. The `redfish` command, in `cmd/redfish`, also
uses cobra.

## Read a node {#read}

A `Client` reads one BMC. The address must be https and must carry no credentials; the credentials are
given apart, through `Credentials`, so a password lives as long as a request.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/go-redfish/redfish"
)

func main() {
	c, err := redfish.New(redfish.Config{
		BaseURL:     "https://bmc.example.org",
		Credentials: redfish.Static{User: "monitor", Password: readSecret()},
		Session:     true, // log in once, send X-Auth-Token, log out on Close
		CAFile:      "/etc/ssl/bmc-ca.pem",
		Timeout:     10 * time.Second,
	})
	if err != nil {
		log.Fatal(err)
	}
	ctx := context.Background()
	defer c.Close(ctx)

	r, err := c.Read(ctx)
	if err != nil {
		log.Fatal(err)
	}
	switch {
	case r.EnergyKWh != nil:
		fmt.Printf("%s: %.3f kWh (counter)\n", r.Chassis, *r.EnergyKWh)
	case r.PowerWatts != nil:
		fmt.Printf("%s: %.0f W\n", r.Chassis, *r.PowerWatts)
	}
}
```

(`readSecret` is yours: read the password from a file or a secret store, never from a command line
or an environment variable.)

`Read` takes the `EnergykWh` counter of the chassis' `EnvironmentMetrics`; with no counter, its power
(`PowerWatts`, else the `PowerControl` of the `Power` resource). The service is discovered once, from
`/redfish/v1`, and the links it gives are remembered. `Config.MetricsPath` skips discovery when you
know the resource; `Config.Chassis` picks one chassis out of several.

## Turn reads into energy {#meter}

A `Meter` turns the successive readings of one BMC into intervals:

```go
var m redfish.Meter // MaxGap 0 means 5 minutes
for range time.Tick(time.Minute) {
	r, err := c.Read(ctx)
	if err != nil {
		continue
	}
	iv, ok := m.Observe(r)
	if !ok {
		continue // the first read is only a baseline
	}
	if !iv.Valid {
		fmt.Println("no energy:", iv.Reason) // never bill 0 as a measurement
		continue
	}
	fmt.Printf("%s %.4f kWh\n", iv.Source, iv.KWh)
}
```

With a counter the interval is the counter's difference, exact even if reads were missed. With only a
power it is the trapezoid of two samples, an estimate between them, and samples further apart than
`MaxGap` are not joined. A counter that goes down (a reboot) gives an invalid interval with the reason
`counter reset`, never a negative energy.

## Any other request {#requests}

The generic methods reach any resource of the service:

```go
r, err := c.Get(ctx, "/redfish/v1/Systems/437XR1138R2")
// change a property only if nobody changed the resource since it was read
_, err = c.Patch(ctx, "/redfish/v1/Systems/437XR1138R2",
	map[string]any{"AssetTag": "rack-12"}, r.ETag())
```

`Post` invokes an action. When the service answers 202 with a task monitor, `WaitTask` polls the monitor
until the task ends. `Do` takes a full `Request` when you need another method or header. To try any of
these without a real machine, see the [mock BMC]({{< relref "mock-bmc.md" >}}).

## Systems {#systems}

Since v0.4.0 the client drives the ComputerSystem resources the way the Ansible modules do:
`SystemPower` and `ManagePower`, `SetBootOverride` and `SetBootOrder`, `BiosAttributes`,
`SetBiosAttributes` and `VerifyBiosAttributes`, `SetSecureBoot`, `SetPowerRestorePolicy`,
`SetIndicatorLED`, `SetAssetTag`, the inventories (`SystemInventory`, `CPUInventory`, `MemoryInventory`,
`NICInventory`, `PSUInventory`) and `SystemHealthReport`. Like the Ansible modules, a power command is
mapped to a `ResetType` the service allows (its `ActionInfo` or `AllowableValues`). Nothing is sent when
the system is already in the state asked for, and an action the resource does not have is
`ErrUnsupported`. With several systems, `ResolveSystem` picks one by id. Without an id, such a service
returns `ErrAmbiguous`. The [`systems` commands]({{< relref "cli.md#systems" >}}) call these functions.

## Errors {#errors}

A refused URL is an `ErrInsecure`. An HTTP status the BMC answered is a `*StatusError`, which carries the
Redfish error's code, message and extended info. Both can be tested with `errors.Is` and `errors.As`.
