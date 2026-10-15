# NETGEAR Genie Integration for Elastic

[![CI](https://github.com/NateShoffner/netgear-genie-elastic-integration/actions/workflows/ci.yml/badge.svg)](https://github.com/NateShoffner/netgear-genie-elastic-integration/actions/workflows/ci.yml)

An Elastic Agent integration that collects the DOCSIS event log (`event_log`),
bonded channel metrics (`channel`) and device status (`status`) from a
[NETGEAR](https://www.netgear.com/) cable modem running the Genie web interface.

The modem offers no API, no SNMP and no syslog export, so all three data streams
sign in to the web interface with the CEL input and scrape the pages. Built and
tested against a **CM1000v2** on firmware `V10.01.07`; other modems in the same
Genie firmware family are likely to work but are untested.

**[Read the integration documentation](packages/netgear_genie/docs/README.md)**
for setup steps, configuration options, exported fields, and troubleshooting.

## Dashboards

Modem health overview: web interface availability, uptime, the DOCSIS startup
procedure, and the event log by severity and event code.

![Modem health overview dashboard](packages/netgear_genie/img/netgear_genie-modem-health-overview.png)

DOCSIS channels: locked channel counts, per channel power and signal to noise
ratio against the levels the DOCSIS specification calls healthy, and codeword
error rates.

![DOCSIS channels dashboard](packages/netgear_genie/img/netgear_genie-docsis-channels.png)

> Rendered from generated data with a placeholder serial number and
> documentation-range MAC addresses, not from a real device.

## Installation

Download the `.zip` from the [latest release](https://github.com/NateShoffner/netgear-genie-elastic-integration/releases/latest)
and upload it through **Kibana > Integrations > Upload integration**, or:

```sh
elastic-package install --zip netgear_genie-<version>.zip
```

To build it yourself instead:

```sh
cd packages/netgear_genie
elastic-package build     # writes build/packages/netgear_genie-<version>.zip
elastic-package install
```

## Development

Requires [`elastic-package`](https://github.com/elastic/elastic-package) and
Docker. Run from `packages/netgear_genie`:

```sh
elastic-package check            # lint and build
elastic-package stack up -d -v   # needed by both test commands below
elastic-package test pipeline    # ingest pipeline tests
elastic-package test system      # end-to-end tests against a mock modem
```

No modem is needed: `_dev/deploy/docker/` serves captured pages and reproduces
the interesting failure modes.

Edit documentation in `_dev/build/docs/README.md`, never in `docs/README.md`,
which is regenerated on every build.

### Releasing

Bump `version` in `packages/netgear_genie/manifest.yml`, add a matching
`changelog.yml` entry, then push a tag:

```sh
git tag v0.2.0 && git push origin v0.2.0
```

CI verifies the tag matches the manifest version, runs the tests, builds the
package, and publishes a GitHub release with the `.zip` attached, a build
provenance attestation, and notes drawn from the changelog entry.

Both CI and the release run on the shared reusable workflows in
[`NateShoffner/elastic-package-workflows`](https://github.com/NateShoffner/elastic-package-workflows).

## License

[Elastic License 2.0](LICENSE).

Not affiliated with NETGEAR or Elastic. NETGEAR and Genie are trademarks of
NETGEAR, Inc.; the package icon is NETGEAR's own logo, used only to identify the
device this integrates with.
