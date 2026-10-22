# asterisk-exporter

A lightweight, **read-only** Prometheus exporter for [Asterisk](https://www.asterisk.org/).
It exposes live call, channel, trunk, voice-quality, ARI/Stasis and internal-health
metrics over an HTTP `/metrics` endpoint, suitable for Prometheus + Grafana.

Works with **chan_sip**, **PJSIP** and **ARI** based Asterisk servers (tested on
Asterisk 18, 20 and 22).
dependencies.

---

## Highlights

- **Read-only & safe** — the exporter never performs a write, originate, reload
  or delete on Asterisk. It only issues read AMI actions / read CLI commands and
  (optionally) HTTP `GET` to ARI.
- **Low footprint** — ~20 MB RAM, <1 % CPU on busy production boxes.
- **Bounded cardinality** — trunks/peers/endpoints are labelled, but one-off IDs
  (channel UNIQUEIDs, per-call data) are never turned into labels, so Prometheus
  stays healthy.
- **Self-healing AMI** — two AMI connections (events + actions). On a dropped or
  stalled connection it reconnects automatically and keeps `asterisk_up` /
  `asterisk_ami_connected` accurate.
- **Hot-reloadable trunk filter** — edit `trunks.txt` and it takes effect on the
  next scrape, no restart.
- **Optional ARI collector** — registered Stasis apps, bridge counts and bridge
  leak detection over ARI HTTP.

---

## Table of contents

- [How it works](#how-it-works)
- [Metrics](#metrics)
- [Build](#build)
- [AMI setup (read-only user)](#ami-setup-read-only-user)
- [ARI setup (optional, read-only)](#ari-setup-optional-read-only)
- [Run](#run)
- [Flags](#flags)
- [The trunk filter (`trunks.txt`)](#the-trunk-filter-trunkstxt)
- [systemd service](#systemd-service)
- [Prometheus scrape config](#prometheus-scrape-config)
- [Grafana dashboard](#grafana-dashboard)
- [Per-server-type recommendations](#per-server-type-recommendations)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)

---

## How it works

The exporter talks to Asterisk over the **AMI** (Asterisk Manager Interface) and,
optionally, over **ARI HTTP**. It keeps two independent AMI connections:

| Connection | Purpose |
|---|---|
| **Event listener** | Subscribes to AMI events (`Newchannel`, `Hangup`, `Bridge`, …) and maintains real-time counters: active calls/channels, call totals, hangup causes, drops. |
| **Action poller** | Periodically runs read AMI actions and CLI commands (`CoreStatus`, `CoreShowChannels`, `SIPpeers`, `PJSIPShowEndpoints`, `BridgeList`, `core show calls`, `sip show channelstats`, `sip show registry`, `core show taskprocessors`, …) to fill gauges. |

Polling happens on two cadences:

- **Light poll** (`-poll-interval`, default 15 s) — cheap actions: core status,
  channel detail, peers/endpoints, bridges.
- **Heavy poll** (`-qos-poll-interval`, default 30 s) — expensive CLI output:
  QoS channel stats, registry, taskprocessors.

If `-ari-url` is set, an **ARI HTTP collector** polls `GET /ari/applications` and
`GET /ari/bridges` on its own interval (`-ari-poll-interval`, default 30 s) to
report registered Stasis apps and (empty) bridge counts.

Every collector is independent: if one command fails (permission denied, module
not loaded, ARI down) it is skipped and logged — the rest keep working and the
exporter stays up.

---

## Metrics

All metrics are prefixed `asterisk_`. Key ones:

### Status & build
| Metric | Type | Description |
|---|---|---|
| `asterisk_up` | gauge | 1 if Asterisk is reachable via AMI |
| `asterisk_ami_connected` | gauge | 1 if the AMI connection is established |
| `asterisk_uptime_seconds` | gauge | Asterisk core uptime |
| `asterisk_last_reload_seconds` | gauge | Seconds since last reload |
| `asterisk_build_info{version,hostname,system_name,kernel}` | gauge | Build/version info (value 1) |

### Calls & channels
| Metric | Type | Description |
|---|---|---|
| `asterisk_active_calls` | gauge | Current active calls (distinct Linkedid, incl. ARI-originated) |
| `asterisk_active_channels` | gauge | Current active channels (one per call leg) |
| `asterisk_core_current_calls` | gauge | Asterisk's own `core show calls` count (dialplan channels only) |
| `asterisk_calls_total` | counter | Total calls (once per call, Uniqueid==Linkedid) |
| `asterisk_calls_answered_total` | counter | Calls that reached Up/answered (ASR approximation) |
| `asterisk_calls_processed_total` | gauge | Asterisk's own processed-calls counter |
| `asterisk_channels_total` | counter | Total channels created |
| `asterisk_calls_hangup_total{cause}` | counter | Hangups by Q.850 cause code |
| `asterisk_channels_by_state{state}` | gauge | Active channels by channel state |
| `asterisk_channels_by_context{context}` | gauge | Active channels by dialplan context |
| `asterisk_long_calls{over}` | gauge | Active calls longer than each threshold (5m/15m/30m) |

### chan_sip
| Metric | Type | Description |
|---|---|---|
| `asterisk_sip_peer_status{sipname,host,status}` | gauge | Peer qualify RTT in ms; negative = problem (UNREACHABLE/LAGGED/…) |
| `asterisk_sip_channel_detail{trunk,state}` | gauge | Active chan_sip channels by trunk and state |
| `asterisk_sip_qos_packet_loss_rx_percent{peer}` / `_tx_` | gauge | Packet loss % per peer |
| `asterisk_sip_qos_avg_jitter_rx_ms{peer}` / `_tx_` | gauge | Average jitter per peer |
| `asterisk_sip_qos_channels_measured` / `_skipped` | gauge | QoS sample size |
| `asterisk_sip_registry_status{host,username,state}` | gauge | Outbound registration (1=Registered, 0=not) |
| `asterisk_sip_registrations_total` / `_registered` | gauge | Configured vs currently-registered outbound trunks |

### PJSIP
| Metric | Type | Description |
|---|---|---|
| `asterisk_pjsip_endpoint_channels{endpoint}` | gauge | Active channels per PJSIP endpoint |
| `asterisk_pjsip_contact_rtt_ms{endpoint,status}` | gauge | Contact qualify RTT (-1 = not reachable) |
| `asterisk_pjsip_qos_packet_loss_rx_percent{endpoint}` / `_tx_` | gauge | Packet loss % per endpoint |
| `asterisk_pjsip_qos_avg_jitter_rx_ms{endpoint}` / `_tx_` | gauge | Average jitter per endpoint |
| `asterisk_pjsip_qos_avg_rtt_ms{endpoint}` | gauge | Media RTT (RTCP) per endpoint |
| `asterisk_pjsip_qos_channels_measured` / `_skipped` | gauge | QoS sample size |

### ARI / Stasis & bridges
| Metric | Type | Description |
|---|---|---|
| `asterisk_ari_up` | gauge | 1 if ARI HTTP answered with valid credentials |
| `asterisk_ari_apps_registered` | gauge | Registered (connected) Stasis apps |
| `asterisk_ari_apps_registered_by_group{group}` | gauge | Apps grouped by `-ari-app-group-regex` (optional) |
| `asterisk_stasis_active_channels` | gauge | Channels currently in a Stasis app |
| `asterisk_bridges{type}` | gauge | Bridges by type (AMI BridgeList) |
| `asterisk_bridges_empty{type}` | gauge | Bridges with zero channels (leak signal) |
| `asterisk_ari_oldest_empty_bridge_age_seconds` | gauge | Age of oldest empty bridge |
| `asterisk_ari_scrape_errors_total` | counter | Failed ARI HTTP polls |

### Internals & exporter health
| Metric | Type | Description |
|---|---|---|
| `asterisk_taskprocessor_in_queue{type}` | gauge | Tasks queued now, by subsystem |
| `asterisk_taskprocessor_max_depth{type}` | gauge | Highest queue depth seen, by subsystem |
| `asterisk_taskprocessor_processed_total{type}` | gauge | Tasks processed, by subsystem |
| `asterisk_exporter_events_processed_total` | counter | AMI events processed |
| `asterisk_exporter_events_dropped_total` | counter | AMI events dropped (buffer full) — should stay 0 |
| `asterisk_exporter_scrape_errors_total` | counter | Failed AMI **action** polls (not events) |

---

## Build

Requires Go 1.23+.

```bash
git clone https://github.com/<you>/asterisk-exporter.git
cd asterisk-exporter
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o asterisk-exporter .
```

This produces a single static binary `asterisk-exporter`.

---

## AMI setup (read-only user)

Add a dedicated **read-only** manager user in `/etc/asterisk/manager.conf`
(or under `manager.d/`). The exporter needs the `system`, `reporting`, `command`
and `call` read classes; nothing else, and **no write permissions**:

```ini
[monitor]
secret = CHANGE_ME_STRONG_SECRET
deny = 0.0.0.0/0.0.0.0
permit = 127.0.0.1/255.255.255.255     ; exporter runs on the box
read = system,reporting,command,call
write =
writetimeout = 5000
```

Then: `asterisk -rx "manager reload"`.

> The `command` class is required because some metrics come from read CLI
> commands (`core show calls`, `sip show channelstats`, `sip show registry`,
> `core show taskprocessors`). The exporter only ever runs **read** CLI commands.
> Leave `write =` empty.

---

## ARI setup (optional, read-only)

Only needed on ARI/Stasis boxes, and only if you want ARI app / bridge-leak
metrics. In `/etc/asterisk/ari.conf`:

```ini
[monitor]
type = user
read_only = yes
password = CHANGE_ME_ARI_SECRET
```

`read_only = yes` is enough — the exporter only issues `GET` requests.
Make sure ARI HTTP is enabled in `http.conf` and note its `bindport`.

---

## Run

chan_sip server:
```bash
./asterisk-exporter \
  -listen 0.0.0.0:9110 \
  -ami-user monitor -ami-pass-file /etc/asterisk_exporter/ami.pass \
  -collect-sip=true -collect-pjsip=false \
  -trunkfile /etc/asterisk_exporter/trunks.txt
```

PJSIP server:
```bash
./asterisk-exporter \
  -listen 0.0.0.0:9110 \
  -ami-user monitor -ami-pass-file /etc/asterisk_exporter/ami.pass \
  -collect-sip=false -collect-pjsip=true
```

Add ARI (any server that runs Stasis):
```bash
  -collect-ari=true \
  -ari-url http://127.0.0.1:8088 \
  -ari-user monitor -ari-pass-file /etc/asterisk_exporter/ari.pass
```

Protect `/metrics` with basic auth by pointing `-authfile` at a
`username:password` file.

---

## Flags

| Flag | Default | Description |
|---|---|---|
| `-listen` | `0.0.0.0:9110` | HTTP listen address for `/metrics` |
| `-authfile` | | File with `username:password` for /metrics basic auth |
| `-ami-host` / `-ami-port` | `127.0.0.1` / `5038` | AMI address |
| `-ami-user` | `monitor` | AMI username |
| `-ami-pass` / `-ami-pass-file` | | AMI password (prefer the file) |
| `-poll-interval` | `15s` | Light AMI action poll interval |
| `-qos-poll-interval` | `30s` | Heavy QoS/CLI poll interval |
| `-event-buffer` | `10000` | Bounded event channel size (drop-on-full) |
| `-trunkfile` | | File of peer names to filter chan_sip peers (hot-reloaded) |
| `-collect-ami-events` | `true` | Real-time AMI event listener |
| `-collect-ami-poll` | `true` | Periodic AMI action poller |
| `-collect-core-active` | `true` | Active call/channel gauges |
| `-collect-core-drops` | `true` | Hangup-cause counters |
| `-collect-sip` | `false` | chan_sip metrics |
| `-collect-sip-peers` | `true` | chan_sip peer status (needs collect-sip) |
| `-collect-sip-qos` | `true` | chan_sip QoS via `sip show channelstats` |
| `-collect-sip-registry` | `true` | Outbound registrations via `sip show registry` |
| `-collect-pjsip` | `true` | PJSIP metrics |
| `-collect-pjsip-endpoints` | `true` | PJSIP endpoints/RTT |
| `-collect-pjsip-qos` | `true` | PJSIP QoS |
| `-collect-bridges` | `true` | Bridge counts + empty bridges (AMI BridgeList) |
| `-collect-taskprocessors` | `false` | Taskprocessor health (needs command class) |
| `-collect-ari` | `false` | ARI/Stasis metrics |
| `-collect-stasis-channels` | `true` | Stasis channel count (needs collect-ari) |
| `-ari-url` | | ARI HTTP base URL; empty disables the ARI collector |
| `-ari-user` | | ARI username (`read_only=yes` is enough) |
| `-ari-pass` / `-ari-pass-file` | | ARI password (prefer the file) |
| `-ari-poll-interval` | `30s` | ARI HTTP poll interval |
| `-ari-app-group-regex` | | Group ARI app names into a bounded `group` label |

---

## The trunk filter (`trunks.txt`)

By default chan_sip peer metrics include **every** peer (extensions too). On a
box with thousands of registrations that is a lot of series. Point `-trunkfile`
at a file with **one peer name per line** to keep only the trunks you care about:

```
BhopalVodaV92
SIPP229
SIPS228
```

Generate it from the live peers that are `OK`:

```bash
asterisk -rx "sip show peers" | grep ' OK ' | awk '{print $1}' | cut -d/ -f1 | sort -u
```

> For a `name/user` peer, use just `name` (the exporter matches the peer name).

The file is **re-read automatically** on each poll (cached by modification
time), so editing it takes effect on the next scrape — no restart needed.

---

## systemd service

`/etc/systemd/system/asterisk-exporter.service`:

```ini
[Unit]
Description=Asterisk Prometheus Exporter
After=network-online.target asterisk.service
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/etc/asterisk_exporter
ExecStart=/usr/local/bin/asterisk-exporter \
  -listen 0.0.0.0:9110 \
  -authfile /etc/asterisk_exporter/auth.txt \
  -ami-host 127.0.0.1 -ami-port 5038 \
  -ami-user monitor \
  -ami-pass-file /etc/asterisk_exporter/ami.pass \
  -collect-sip=true -collect-pjsip=false \
  -trunkfile /etc/asterisk_exporter/trunks.txt
Restart=always
RestartSec=5
MemoryMax=512M
CPUQuota=50%
LimitNOFILE=65536
NoNewPrivileges=yes
ProtectSystem=full
ProtectHome=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now asterisk-exporter
```

Put the secrets in `/etc/asterisk_exporter/` (mode `0600`): `auth.txt`
(`user:pass` for /metrics), `ami.pass`, `ari.pass`.

---

## Prometheus scrape config

```yaml
scrape_configs:
  - job_name: asterisk
    scrape_interval: 30s
    basic_auth:
      username: prometheus
      password: CHANGE_ME_metrics_pass
    static_configs:
      - targets:
          - 10.0.0.211:9110
          - 10.0.0.212:9110
        labels:
          group: sip
```

---

## Grafana dashboard

Import `asterisk-grafana-dashboard.json` (Dashboards → New → Import). It has a
**Job** and **Server** variable and panels for overview, calls & channels,
trunks, voice quality, ARI/Stasis and internals. Select your Prometheus
datasource, job and server after import.

---

## Per-server-type recommendations

| Server | Suggested flags |
|---|---|
| **chan_sip** | `-collect-sip=true -collect-pjsip=false -trunkfile … -collect-taskprocessors=true` |
| **PJSIP** | `-collect-sip=false -collect-pjsip=true -collect-taskprocessors=true` |
| **ARI/Stasis** | the above for its channel driver, plus `-collect-ari=true -ari-url … -ari-user … -ari-pass-file …` |

Do **not** set `-ari-url` on servers that don't run ARI — it just logs
`connection refused` each poll.

---

## Troubleshooting

- **`asterisk_up` missing / 401 on curl** — `/metrics` has basic auth; use
  `curl -u user:pass http://127.0.0.1:9110/metrics`.
- **`asterisk_exporter_scrape_errors_total` slowly increasing** — these are AMI
  **action** poll errors (not events). A few per day are normal (periodic
  reconnect). Events are healthy as long as
  `asterisk_exporter_events_dropped_total` stays 0.
- **ARI `connection refused`** — ARI HTTP isn't up on that port. Check
  `asterisk -rx "http show status"` (bindport) and `ari.conf`.
- **Peers/registrations not showing after editing trunks.txt** — they appear on
  the next scrape automatically; no restart needed.
- **High-cardinality / huge bridge counts** — if a box is leaking bridges
  (`asterisk_bridges_empty` growing for days), that is an ARI-app bug on the
  Asterisk side, not the exporter. On such a box you can reduce load with
  `-collect-bridges=false` and a larger `-qos-poll-interval`.

---

## Security notes

- The exporter is **read-only**: it issues read AMI actions, read CLI commands,
  and ARI `GET` only. It never originates calls, writes config, reloads, or
  deletes anything. Give the AMI user **no write permissions** and the ARI user
  `read_only = yes`.
- Keep `/metrics` behind basic auth (`-authfile`) and/or bind it to an internal
  interface.
- Store AMI/ARI passwords in `*-pass-file` files (mode `0600`), not on the
  command line, so they don't show up in `ps` / `htop`.

---

## License

MIT (or your choice).
