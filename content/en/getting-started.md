---
title: "Getting started"
weight: 20
description: "Install the redfish module, read what a node consumes, and turn reads into energy."
---

## Install {#install}

```sh
go get github.com/go-redfish/redfish@v0.2.0
```

The module has no dependency outside the standard library.

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

## Errors {#errors}

A refused URL is an `ErrInsecure`; an HTTP status the BMC answered is a `*StatusError`. Both can be
tested with `errors.Is` and `errors.As`.
