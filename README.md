# docs

Documentation of [go-redfish](https://github.com/go-redfish), built with Hugo and the
[tannevaled/hextra](https://github.com/tannevaled/hextra) fork (a submodule pinned at the release tag
`v0.24.1`; the page-history tool in CI is the nested module `cmd/pagehistory` at its own tag, `v0.21.0`).

Published at <https://go-redfish.github.io/docs/>:

- **Overview** of the organisation.
- **Getting started**: install, read a node, turn reads into energy.
- **Security model**: what the client refuses and bounds.
- **Mock BMC** and **Command-line reference**: stubs, marked as not released.
- **Parity**: the status against the Python tooling (the table itself lives in `redfish/PARITY.md`).

Each version of the documentation is a separate build under `docs/<version>/` on the `gh-pages`
branch; `main` is the version while the pages describe more than the last tag.

```sh
git clone --recurse-submodules https://github.com/go-redfish/docs
hugo server
```

Build as CI does: `hugo --gc --minify --panicOnWarning --printPathWarnings`.
