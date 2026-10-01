# NETGEAR Genie Integration for Elastic

[![CI](https://github.com/NateShoffner/netgear-genie-elastic-integration/actions/workflows/ci.yml/badge.svg)](https://github.com/NateShoffner/netgear-genie-elastic-integration/actions/workflows/ci.yml)

An Elastic Agent integration that collects the DOCSIS event log (`event_log`),
the bonded channel table (`channel`) and the device status (`status`) from a
[NETGEAR](https://www.netgear.com/) cable modem running the Genie web
interface.

The modem offers no API, no SNMP and no syslog export, so all three data streams
sign in to the web interface with the CEL input and scrape the pages a browser
would show you. Built and tested against a **CM1000v2** on firmware `V10.01.07`;
other modems in the same Genie firmware family are likely to work but are
untested.

**[Read the integration documentation](packages/netgear_genie/docs/README.md)**
for setup steps, configuration options, exported fields, and troubleshooting.

## Why

A cable modem is normally a black box. When the connection drops for ten minutes
at 3am there is nothing to look at afterwards, because the only record lives on
a web page the modem overwrites as it runs.

This integration keeps that record, so the channel power, signal to noise ratio
and codeword error counts that explain the outage are still there in the
morning. It also records the health of the web interface itself: polls that fail
become documents rather than gaps, which is what makes it possible to alert on a
modem that has stopped answering.

That matters more than it sounds. This firmware family can wedge into a state
where the web interface accepts a login, redirects as though it succeeded, and
then serves the login page for every request, indefinitely. The modem keeps
passing traffic throughout, so nothing else notices and no ordinary reachability
check catches it. The `status` data stream records it as
`netgear_genie.status.web_ui.state: session_stuck`.

## Dashboards

**Modem health overview**: web interface availability, time since the last
reboot, the DOCSIS startup procedure, modem uptime, and the event log by
severity and event code.

![Modem health overview dashboard](packages/netgear_genie/img/netgear_genie-modem-health-overview.png)

**DOCSIS channels**: locked channel counts, per channel power and signal to
noise ratio against the levels the DOCSIS specification calls healthy, and
codeword error rates. Dropdowns filter by direction, modulation, lock status and
channel ID.

![DOCSIS channels dashboard](packages/netgear_genie/img/netgear_genie-docsis-channels.png)

Both dashboards link to each other from a bar at the top, carrying the current
time range and filters across.

> Rendered from generated data with a placeholder serial number and
> documentation-range MAC addresses, not from a real device.

Two alerting rule templates ship alongside them, covering two failures where
neither one covers the other: the modem failing while collection still works,
and collection itself failing, which no query over the modem's own data can
detect.

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

Then add the integration to an agent policy and set the modem URL (usually
`http://192.168.100.1`, which the modem answers on even in bridge mode) and the
web interface credentials.

## Development

Requires [`elastic-package`](https://github.com/elastic/elastic-package) and
Docker. Run from `packages/netgear_genie`:

```sh
elastic-package check            # lint and build
elastic-package stack up -d -v   # needed by both test commands below
elastic-package test pipeline    # ingest pipeline tests
elastic-package test system      # end-to-end tests against a mock modem
```

Edit documentation in `_dev/build/docs/README.md`, never in `docs/README.md`,
which is regenerated on every build.

No modem is needed to develop against. `_dev/deploy/docker/` runs
[elastic/stream](https://github.com/elastic/stream) as a stand-in that serves
captured pages and reproduces the interesting failure modes: the single-session
conflict, the wedged web server, HTTP errors and an unreadable login redirect.

CEL programs can be run offline with [mito](https://github.com/elastic/mito).
Note that mito follows HTTP redirects and Filebeat's CEL input does not behave
identically, so a program passing under mito is necessary but not sufficient.

### Releasing

Bump `version` in `packages/netgear_genie/manifest.yml`, add a matching
`changelog.yml` entry, then push a tag:

```sh
git tag v0.2.0 && git push origin v0.2.0
```

CI verifies the tag matches the manifest version, builds the package, and
publishes a GitHub release with the `.zip` attached and notes drawn from the
changelog entry.

## License

[Elastic License 2.0](LICENSE).

This project is not affiliated with, endorsed by, or supported by NETGEAR or
Elastic. NETGEAR and Genie are trademarks of NETGEAR, Inc. The package icon is
NETGEAR's logo, taken from the web interface of the modem the integration reads,
and is used only to identify the device this integrates with.
