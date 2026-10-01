{{- generatedHeader }}

# NETGEAR Genie

## Overview

The NETGEAR Genie integration collects the event log, the DOCSIS
channel table and the device status from a NETGEAR cable modem that serves the
Genie web interface, and turns them into logs, metrics and dashboards.

A cable modem is normally a black box: when the connection degrades there is
nothing to look at afterwards, because the only record lives on a web page that
is overwritten as the modem runs. This integration polls that web interface on a
schedule and keeps the result, so a problem that happened overnight is still
there in the morning, with the channel power, signal to noise ratio and codeword
error counts that go with it.

The integration collects:

- **Event log**: the modem's DOCSIS event table, one document per logged event,
  with the severity, the DOCSIS event code and the text split out.
- **DOCSIS channels**: the downstream and upstream bonded channel table, one
  document per channel per poll, with lock status, frequency, power, signal to
  noise ratio and codeword counters.
- **Modem status**: the DOCSIS startup procedure, the device identity, the
  modem's uptime, and the reachability of the web interface itself.

### How it works

The modem has no API, no SNMP and no syslog export, so the integration signs in
to the web interface the same way a browser does and scrapes the pages. Each
poll fetches the login page for its one-time token, posts the credentials, and
then reads the page it needs.

The modem allows only **one web interface session at a time**. While the
integration holds the session, signing in from a browser takes it away, and the
next poll takes it back. See [Take over the modem web session](#settings) below.

### Health of the web interface itself

Every poll of the **Modem status** data stream writes a document, including the
polls that fail. A poll that could not reach the modem, could not sign in, or
was answered with the login page instead of the page it asked for is recorded
with `netgear_genie.status.web_ui.state` set to one of:

| State | Meaning |
| --- | --- |
| `ok` | The web interface answered and the status page was collected. |
| `unreachable` | The modem did not answer at all. |
| `http_error` | The modem answered with an HTTP error. |
| `auth_failed` | The modem rejected the credentials. |
| `session_conflict` | Another device holds the modem's single web session. |
| `session_stuck` | The modem accepted the sign in but still answers protected pages with the login page. |

`session_stuck` is worth calling out. This firmware family can get into a state
where the web interface accepts a login, redirects as though it succeeded, and
then serves the login page for every request, indefinitely. The modem keeps
passing traffic the whole time, so nothing else notices. A failed poll becoming
a document rather than a gap is what makes that state visible, and what lets an
alert fire on it.

## Compatibility

Developed against a NETGEAR CM1000v2 on firmware `V10.01.07`. Other NETGEAR
cable modems that serve the same Genie web interface are likely to work, since
the pages and the sign-in flow are shared across the firmware family, but they
have not been tested.

This integration reads the modem's web interface over plain HTTP. The modem does
not offer HTTPS, SSH or SNMP.

## Requirements

- An Elastic Agent that can reach the modem's web interface over HTTP. The
  modem is usually at `http://192.168.100.1`, which it answers on even when it
  is in bridge mode and the rest of the network is behind a router.
- The modem's web interface username and password. NETGEAR ships these as
  `admin` / `password`; change them on the modem if you have not already.

## Setup

1. In Kibana, go to **Management > Integrations**, search for **NETGEAR
   Genie** and select **Add NETGEAR Genie**.
2. Set **Modem URL**, **Username** and **Password**.
3. Enable the data streams you want and adjust their poll intervals.
4. Assign the policy to the agent that can reach the modem.

### Settings

| Setting | Default | Notes |
| --- | --- | --- |
| Modem URL | `http://192.168.100.1` | Base URL of the web interface. |
| Username / Password | `admin` | Web interface credentials. The password is stored as a secret. |
| HTTP Client Timeout | `30s` | The modem's web server is slow; several seconds per page is normal. |
| Interval | `5m` | Per data stream. See the note below before lowering it. |
| Take over the modem web session | | The modem permits one session at a time. When enabled, a poll that finds another device holding the session takes it over, which signs that device out. Leave it off if you want the integration to stand aside and report `session_conflict` instead. |
| Time zone offset | `UTC` | The modem reports event times with no time zone. Set this to the modem's local time zone if it is not UTC, or event timestamps will be shifted. |

**Choosing an interval.** Each poll is a full sign-in plus a page fetch against
a small embedded web server, and each enabled data stream polls independently.
Five minutes per data stream is a reasonable default. Polling much more often
gives the modem little time to do anything else, and because the modem allows
only one session, it also means a browser session is being displaced more often.

## Data streams

### Event log

The modem's DOCSIS event table, from `/EventLog.asp`. Rows are aggregated in
place by the modem: a repeating event keeps one row and bumps its occurrence
count and last-seen time rather than adding a new row. The integration emits a
row when it is new or when its count or last-seen time changes, and gives each
one a deterministic document ID, so a row that is collected twice is written
over itself instead of duplicated.

Events the modem logged before it had learned the time of day carry the literal
`Time Not Established` instead of a timestamp. Those documents keep their
collection time as `@timestamp` and are tagged `time_not_established`, so a
stand-in timestamp can be told apart from a real event time.

{{event "event_log"}}

{{fields "event_log"}}

### DOCSIS channels

The downstream and upstream bonded channel table, from `/DocsisStatus.asp`. One
document per channel per poll.

The codeword counters (`unerrored_codewords`, `correctable_codewords`,
`uncorrectable_codewords`) are cumulative totals that the modem resets when it
reboots. Chart the change between polls rather than the raw value.

{{event "channel"}}

{{fields "channel"}}

### Modem status

The DOCSIS startup procedure, device identity and uptime, from
`/DocsisStatus.asp` and `/RouterStatus.asp`, plus the health of the web
interface described above. Exactly one document per poll, always.

{{event "status"}}

{{fields "status"}}

## Dashboards

**Modem health overview** is the place to start: web interface availability,
time since the last reboot, the DOCSIS startup procedure, modem uptime, and the
event log broken down by severity and event code.

![Modem health overview dashboard](../img/netgear_genie-modem-health-overview.png)

**DOCSIS channels** covers the bonded channels: locked channel counts, per
channel power and signal to noise ratio against the levels the DOCSIS
specification calls healthy, and codeword error rates.

![DOCSIS channels dashboard](../img/netgear_genie-docsis-channels.png)

Both dashboards link to each other from a bar at the top, which carries the
current time range and filters across. The DOCSIS channels dashboard has
dropdowns for direction, modulation, lock status and channel ID.

## Alerts

The integration ships two rule templates, which appear in Kibana under
**Stack Management > Rules** when you create a rule from a template. They cover
two different failures, and neither one covers the other:

- **Modem web interface not responding** fires when a window contains polls but
  none of them healthy, and carries the observed state and reason into the
  alert. This is the modem failing while collection is still working.
- **Modem stopped reporting** fires when no status document arrives at all.
  That is collection failing, which no query over the modem's own data can see,
  because there is nothing to query.

Both templates assume the default five minute interval and use a sixteen minute
window, so three consecutive polls have to fail before they fire. Widen the
window if you raise the interval. Neither template has an action attached;
connect one so the alert reaches you.

## Troubleshooting

**Everything reports `session_stuck` and will not recover.** The modem's web
server has wedged. It keeps passing traffic, so the internet still works, but no
page will load for anyone, including a browser. Power cycling the modem clears
it. It can also clear on its own after the modem re-registers with the CMTS.

**Polls report `session_conflict`.** Another device is signed in to the modem.
Either sign that session out, or enable **Take over the modem web session**.

**Event timestamps are hours off.** The modem reports event times with no time
zone. Set **Time zone offset** on the data stream to the modem's local time
zone.
