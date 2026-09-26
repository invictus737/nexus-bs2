# Nexus-BS 2.0 configuration manual (`nexus2.toml`)

Translations: [Română](Manual_conf.ro.md) · [Deutsch](Manual_conf.de.md) · [Español](Manual_conf.es.md) · [Português](Manual_conf.pt.md)

This manual explains every key of the station configuration file, how the file
is checked and how changes reach the running station. It was written from the
source code (commit `d7fdec7` and later), not from memory. Where the code does
not settle a detail, the manual says so instead of guessing.

The working station configuration is the reference for recommended values. The only
example shipped with Nexus-BS 2.0, `examples/nexus2.example.toml`, uses the same settings with generic
identities (`N0CALL`, `CHANGE_ME`), the TETRA BlueStation example frequency plan and TX inhibited. In the tables below:

- **Default** is what the code uses when the key is absent. "required" means there is no default.
- **Station** is the value in the working station file. A dash means the key is absent there.

Contents:
1. [How the file is read](#1-how-the-file-is-read)
2. [Applying a change](#2-applying-a-change)
3. [Frequencies and the channel plan](#3-frequencies-and-the-channel-plan)
4. [Transmit permission and the RF switch](#4-transmit-permission-and-the-rf-switch)
5. [`[sdr]`](#5-sdr-the-shared-radio)
6. [`[[channel]]`](#6-channel-common-keys)
7. [`[channel.access]`](#7-channelaccess-admission-policy)
8. [TETRA](#8-tetra)
9. [DMR](#9-dmr)
10. [P25](#10-p25)
11. [Analog FM](#11-analog-fm)
12. [Deferred modes](#12-deferred-modes)
13. [`[dashboard]`, `[network]`, `[touch]`](#13-dashboard-network-touch)
14. [Command line](#14-command-line)
15. [Recipes](#15-recipes)
16. [Known limitations and open points](#16-known-limitations-and-open-points)

---

## 1. How the file is read

- **Format.** The file is TOML 1.0. Within a table, the order of keys is free. A table header such as
  `[network.brew]` may appear anywhere in the file, even after the `[[channel]]` entries. The station file uses this
  to group each network link with its mode.
- **Strict.** Every table rejects unknown keys. A typo is a hard error, for example
  `config: unknown field `brightnes` near line 70`. Error messages give the key path and line number, never the
  value, so secrets do not leak into logs.
- **Required tables.** `[sdr]`, at least one `[[channel]]`, `[dashboard]` and `[network]` are required.
  `[dashboard]` and `[network]` may be empty. `[touch]` and `[callsigns]` are optional.
- **Value forms.**
  - An integer is accepted where a decimal is expected (`rx_gain_db = 30`).
  - Hex, octal and binary literals work for any integer (`nac = 0x293`), and so do underscores (`offset_hz = -50_000`).
  - Negative numbers must be written in decimal.
- **Units.** Frequencies are in Hz. Times are in ms unless the key name ends in `_secs`.
- **Secrets.** Secrets are plain strings: `dashboard.auth.password`, `network.brew.password`,
  `channel.dmr.network.password` and `sdr.uri`. They are hidden from debug output and from the redacted export.
  Keep the file mode 0600. The dashboard and the management API both write it that way.

## 2. Applying a change

**The radio process reads the file once, when it starts.** Editing the file changes nothing until the service
restarts. There are two exceptions:

| What | Takes effect |
|---|---|
| `[touch]` | The panel re-reads it within about 5 s. An out-of-range value falls back to its default. |
| Enabling or disabling a **DMR, P25 or FM** channel from the dashboard or panel toggle | Immediately, without a restart. It is held **in memory only** and not written to the file: after a service restart the file's `enabled` value applies again. TETRA has no live toggle. |

Everything else needs a restart. `[callsigns]` is also loaded only at start.

**Check before applying:**

```sh
nexus2 --config nexus2.toml plan         # frequency plan, no hardware
nexus2 --config nexus2.toml doctor       # full static validation
nexus2 --config nexus2.toml doctor --live   # plus the checks a live start makes
```

`doctor` never opens the SDR or the network. It does not resolve DNS, bind sockets or read the `[callsigns]`
database, so a clean `doctor` does not guarantee that the network links come up.

**On the station** (management API from a workstation; see MANAGEMENT-API.md):

```sh
A="python3 scripts/nexus2-admin.py --url https://<station-ip>:9443 --connect-ip <station-ip>"
$A config > current.json                 # contains secrets: keep private
$A validate candidate.toml               # validate with the station's own checks
$A set-config candidate.toml --if-match <config_sha256 from status>
```

- `set-config` swaps the file, keeps a backup, restarts the radio and rolls back if the process does not stay
  healthy.
- The hash precondition refuses to overwrite a file that changed in the meantime.
- The restart briefly interrupts **every mode**.
- The manager validates with its own binary. After a radio upgrade that adds new keys, upgrade the manager too,
  otherwise it rejects them (HTTP 422).
- While the management API is installed (marker file `/opt/nexus-bs2/.management-enabled`), the dashboard's own
  config editor and settings form refuse to save and answer `use_management_api`.

## 3. Frequencies and the channel plan

One SDR carries every carrier. Each channel is an offset from the two local oscillators:

```
RX frequency (uplink, terminals -> station)   = sdr.rx_lo_hz + offset_hz
TX frequency (downlink, station -> terminals) = sdr.tx_lo_hz + offset_hz
```

There is one `offset_hz` per channel, so every carrier has the same duplex split: `tx_lo_hz - rx_lo_hz`.

**Plan rules.** These are checked for **all** channels, disabled ones included:

| Rule | Error if broken |
|---|---|
| 1 to 7 channels in total | channel count |
| `offset_hz` is an exact multiple of 25 000 | `channel[i].offset_hz: must be an exact multiple of 25000` |
| Any two channels at least 50 000 Hz apart (one empty 25 kHz cell between carriers) | plan error |
| `abs(offset_hz) + 20 000 < sample_rate / 2` | "insufficient Nyquist/filter margin" |
| SX1255 boards (`sx`, `mucell`): no **enabled** channel at `offset_hz = 0`, because it would sit on DC | "carrier would sit on DC; shift rx_lo_hz/tx_lo_hz by a 25 kHz multiple instead" |

- The required analog passband is `2 × (max |offset_hz| + 20 000)`. For the station that is 2 × (175 000 + 20 000)
  = 390 kHz.
- **Sample rate.** When `sdr.sample_rate` is absent or 0, the station asks the radio which rates it supports. It keeps
  those that are at most 1 MHz, a multiple of 50 kHz and wider than the required passband. It then takes the smallest
  one of at least twice the passband, otherwise the widest usable one. The station ends up at 600 kS/s.
- **TETRA frequency encoding.** A TETRA carrier must also be a standard TETRA channel:
  - The TX frequency must sit on the 25 kHz raster, optionally shifted by +6.25, −6.25 or +12.5 kHz.
  - The RX–TX spacing must be one of the standard duplex spacings for the band. In the 400 MHz band these are 10, 7, 8,
    5 or 9.5 MHz. The station uses 7 MHz.
  - Otherwise the error is `tetra: frequencies have no supported standard repeater encoding`.
  - `plan` does not check this; `doctor` and a live start do.

**Station plan** (LOs 431.2875 / 438.2875 MHz):

| Channel | offset_hz | RX MHz | TX MHz | State |
|---|---|---|---|---|
| tetra-data | −50 000 | 431.2375 | 438.2375 | disabled |
| dmr | +25 000 | 431.3125 | 438.3125 | enabled |
| tetra | +75 000 | 431.3625 | 438.3625 | enabled |
| p25 | +125 000 | 431.4125 | 438.4125 | enabled |
| analog (FM) | +175 000 | 431.4625 | 438.4625 | enabled |

To move the whole station, change both LOs by the same amount. To move one carrier, change its `offset_hz`, keep the
grid rules and check with `nexus2 plan`.

## 4. Transmit permission and the RF switch

Three separate things decide whether the station transmits:

1. **`sdr.tx_inhibited`** (config, code default `true`). When true, no transmit path is prepared at all. The station
   file sets `false`. `nexus2 init` always writes `true`.
2. **`--allow-rf-tx`** (command line of `run --live`). This is only process permission. A file with
   `tx_inhibited = false` is refused at start without it. The service unit passes it.
3. **The global RF switch** (RF KILL / RESTORE RF on the dashboard and panel). This is **not a config key**.
   - Its state is kept in `~/.local/state/nexus2/rf-<hash of the config path>`, so it survives restarts. OFF stops RX
     and TX of every mode.
   - Only an operator RF KILL, or a fatal failure to stop the radio cleanly, stores OFF.
   - A failed start keeps ON and retries by itself, after 2 s and then with a growing pause up to 60 s. An example is a
     network name that does not resolve yet during boot.
   - This is a software control, not a hardware interlock.

## 5. `[sdr]`: the shared radio

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `driver` | `"auto"` | — | SoapySDR driver name, or `auto`/empty to use the best radio found. It is **not** used to open the device: `uri` is. It only tells the station whether the board is an SX1255 (`sx`, `mucell`). That affects the DC rule, the default `tx_lead_quanta` and a warning about the fitted HAT. Auto preference order: SX1255 boards (`sx` before `mucell`), Lime, USRP B2xx, other USRP, Pluto, anything else. |
| `uri` | `"auto"` | — | SoapySDR device arguments (1–256 characters), or `auto` to take them from the radio found. |
| `sample_rate` | `0` (auto) | — | Complex sample rate of the whole device. Must be a positive multiple of 50 000 and at most 1 000 000. See §3. |
| `rx_lo_hz` | required | 431287500 | RX local oscillator. |
| `tx_lo_hz` | required | 438287500 | TX local oscillator. |
| `rx_gain_db` | required | 30.0 | Overall RX gain (−100…100), applied before the element gains. |
| `tx_gain_db` | required | 0.0 | Overall TX gain (−100…100). |
| `rx_gains_db.<element>` | none | LNA 42, PGA 16 | Per-element RX gains, applied after the overall gain. Element names come from the driver. |
| `tx_gains_db.<element>` | none | DAC 9, MIXER 30 | Per-element TX gains. |
| `rx_bandwidth_hz` / `tx_bandwidth_hz` | auto | — | Analog filter width. If set, it must be **greater than** the required passband (§3). When absent the driver's range is used. |
| `rx_channel` / `tx_channel` | 0 | — | Hardware channel index on multi-channel radios. |
| `rx_antenna` / `tx_antenna` | per board | — | Antenna port. The defaults are SX1255 `RX`/`TX`, Lime USB `LNAL`/`BAND1`, other Lime `LNAW`/`BAND2`, Pluto `A_BALANCED`/`A`, USRP `TX/RX`. |
| `rx_stream_args` / `tx_stream_args` | empty | — | Extra SoapySDR stream arguments (`key = "value"`). |
| `settings.<key>` | empty | — | Device-wide SoapySDR settings. SX1255 boards get `PA = "AUTO"` unless set here. |
| `ppm` | 0.0 | — | Frequency correction for both paths (−100…100). A nonzero value needs a driver that supports correction, otherwise the radio fails to open. |
| `tx_lead_quanta` | 8 on SX1255, 3 otherwise | — | How many 1 ms blocks of TX samples are prepared ahead of the radio clock (1–8). 8 is measured on the SX1255; 3 is not measured on USB radios. Raise it first if the log shows "TX late block skipped". |
| `sx1255_rx_pll_bw_khz` | driver (300) | — | SX1255 RX PLL loop bandwidth: 75, 150, 225 or 300. Rejected on other radios. |
| `sx1255_tx_pll_bw_khz` | driver (150) | 300 | SX1255 TX PLL loop bandwidth. 300 gives the lowest close-in phase noise on the carrier. |
| `tx_inhibited` | **true** | false | See §4. |
| `io_diagnostics` | false | false | Measurement aid: RX level windows and I/O timing counters, plus the dashboard `rf_io` block. It writes about 11 journal lines per second, so leave it off in normal operation. It does not alter RF. |
| `pipeline_diagnostics` | false | — | Experimental queue and channel instrumentation. |

## 6. `[[channel]]`: common keys

Each `[[channel]]` block is one carrier. Its mode table must follow it, together with `[channel.access]` for
DMR, P25 and FM.

| Key | Default | Meaning and rules |
|---|---|---|
| `id` | required | Unique name, 1–64 characters from `A-Z a-z 0-9 - _`. The dashboard, the control API and a packet-data `cell` refer to channels by it. |
| `mode` | required | `tetra`, `dmr`, `p25` or `fm`. `dstar`, `ysf`, `nxdn` and `pocsag` also parse, but a live start refuses them if enabled (§12). The matching table (`[channel.tetra]`, …) is required and any other mode table is an error. |
| `enabled` | true | Administrative on/off. A disabled channel still counts toward the plan rules and the 7-carrier limit. |
| `offset_hz` | required | Offset from both LOs (§3). |

**Order of `[[channel]]` blocks.** The order does not change any frequency. It sets the index a channel is shown
under in error messages (`channel[2].…`) and in the dashboard settings form.

- The TETRA transmit level is taken from the first enabled TETRA channel that has a `[channel.tetra.worker]`
  (`tx_peak`). Keep the primary TETRA channel first.
- USRP port conflicts are checked against earlier channels only.
- Reordering, adding or removing channels needs a restart.

**Live-start requirements.** These are beyond what `doctor` checks without `--live`:

- Exactly one enabled primary TETRA channel, so the station does not run without TETRA.
- Every other enabled TETRA channel must be that cell's packet-data carrier.
- An enabled DMR channel needs `worker`, `network` and `network.runtime`.
- An enabled P25 channel needs `worker` and `reflector`.
- An enabled FM channel needs `modem`, `worker` and either `svxlink` or `usrp`.
- Every enabled DMR, P25 or FM channel needs `[channel.access]`.

## 7. `[channel.access]`: admission policy

This table decides what traffic a mode lets through. Nothing is admitted by default and it never grants physical TX.
`mode` must equal the channel's mode. TETRA has no access table.

| Mode | Key | Meaning |
|---|---|---|
| dmr | `rf_slots = [TS1, TS2]` (required) | Admit RF traffic (voice, data, CSBK) on each timeslot. |
| dmr | `network_slots = [TS1, TS2]` (required) | Admit network traffic toward RF on each timeslot. |
| dmr | `protected_voice` (default false) | Admit voice carrying the privacy/encryption (PI) flag. When false it is refused in both directions. |
| p25 | `rf` (required) | Admit RF calls. |
| p25 | `network` (required) | Admit reflector traffic toward RF. |
| fm | `selected` (required) | The MMDVM "FM mode selected" input of the repeater logic. It is not the carrier/CTCSS access. |

## 8. TETRA

### 8.1 `[channel.tetra]`

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `mcc` | required | 901 | Mobile country code (0–1023). |
| `mnc` | required | 9999 | Mobile network code (0–16383). |
| `location_area` | required | 2 | Base location area (0–16383). |
| `colour_code` | required | 1 | Colour code (0–63). |
| `rotate_location_area_on_start` | true | — | When true, each start broadcasts a different LA derived from `location_area`, so camped terminals re-register after a restart. Set false to broadcast `location_area` exactly. |
| `implicit_group_affiliation` | true | — | A call or floor request from a terminal not known as attached to that group attaches it instead of being refused. |
| `station_name` | `"Nexus-BS 2.0"` | — | Text of the periodic Home Mode Display message shown on terminals. Empty turns it off. |
| `timezone` | host time zone | — | IANA zone name (e.g. `Europe/Bucharest`) for the cell's clock broadcast. Empty turns the clock off. An invalid name fails `doctor`. |
| `role` | `"primary"` | — / `packet_data` | `packet_data` makes this a secondary data carrier (§8.4). |
| `cell` | none | — / `tetra` | Only for `role = "packet_data"`: the `id` of the primary channel. |

### 8.2 `[channel.tetra.packet_data]`: IP over TETRA (primary only)

When present, the cell offers SNDCP packet data. Terminals get addresses from the pool and reach the station through
a Linux tun interface.

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `ipv4_pool` | required | `10.44.0.0/24` | CIDR with prefix /16…/30 and no host bits. The first host is the gateway on the tun interface (10.44.0.1); the rest are handed to terminals. |
| `tun` | `"tetra0"` | — | Name of the tun interface (1–15 characters). |
| `mtu` | 576 | — | N-PDU size and interface MTU (128–1500). |
| `header_compression` | true | false | Grant RFC 1144 (Van Jacobson) header compression when a terminal asks for it. The Motorola MXP600 drops the activation when it is granted, so the station keeps it off. |
| `max_slots` | 4 | — | Most timeslots one terminal may use on the data carrier. Values outside 1–4 are **silently clamped**, not rejected. |
| `advertise` | true | — | Advertise "SNDCP available" in the system information. When off the service still answers terminals that try. |
| `advertise_advanced_link` | true | — | Advertise "advanced link supported". |

### 8.3 `[channel.tetra.worker]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `tx_peak` | 0.5 | 0.8 | Peak baseband envelope of the TETRA carrier, on the same scale as DMR, P25 and FM `tx_peak`. 0.5 gives the same peak as DMR; about 0.79 gives the same average power, with peaks 4 dB higher. Keep it at 0.8 or below. The code only rejects values outside 0–1.0, and only when TX is enabled. |

### 8.4 Packet-data carrier (`role = "packet_data"`)

A second TETRA carrier of the same cell, with packet data on all four timeslots and no control channel. Terminals are
sent to it by channel allocation from the primary. Rules:

- `cell` must name an existing primary TETRA channel, and that primary must have `[channel.tetra.packet_data]`.
- `mcc`, `mnc`, `location_area` and `colour_code` must equal the primary's.
- A packet-data carrier has no `packet_data` table of its own, and there is at most one per primary.
- Its frequencies must be in the primary's band with the same duplex spacing.
- When it is disabled (`enabled = false`), the cell keeps packet data on the primary carrier only.

### 8.5 `[network.brew]`: TETRA core link (Brew / TetraPack)

This table requires at least one enabled TETRA channel.

| Key | Default | Station | Meaning |
|---|---|---|---|
| `host` | required | core.tetrapack.online | Server host name or IP. |
| `port` | 443 | 443 | Server port. |
| `tls` | true | true | Use TLS (wss/https). |
| `username` / `password` | none | set | HTTP Digest login. Either both or neither. |
| `reconnect_delay_secs` | 15 | 15 | Pause before reconnecting (1–300). |
| `jitter_initial_latency_frames` | 0 | — | Extra initial playout delay, in frames, for network voice. |
| `feature_sds_enabled` | true | — | Pass SDS messages between local and network terminals. |
| `feature_rssi_export` | false | — | Report RSSI to the server. |
| `whitelisted_ssis` | none | — | Only allow network calls with these remote SSIs (up to 4096 entries, each at most 0xFFFFFF). |

The legacy keys `network.brew_endpoint` and `network.brew_token` are still parsed, but `doctor` and a live start
reject them. Use `[network.brew]` instead.

## 9. DMR

### 9.1 `[channel.dmr]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `id` | required | 1234567 | 24-bit repeater ID on RF (1–16777215). It is also the network ID unless `network.id` is set. |
| `colour_code` | required | 1 | Colour code (0–15). |

### 9.2 `[channel.dmr.worker]`: native modem (every key required)

| Key | Station | Meaning and rules |
|---|---|---|
| `symbol_deviation` | 10.0 | FM scaling of the 4FSK symbols, used for RX conditioning and TX. Must be > 0. |
| `tx_level_q15` | −13056 | Signed Q15 TX deviation gain. −13056 = −102 × 128, the reference SDR level. A negative value inverts the deviation. |
| `tx_peak` | 0.5 | Peak baseband envelope, 0 < x ≤ 0.8. |
| `power_calibration` | 0 | Offset added to the RSSI dB reading. |
| `receiver_delay` | 3 | Per-slot delay of the repeater receiver. Its exact unit is not documented in this code base; keep 3. |
| `marker_offset_numerator` / `marker_offset_denominator` | 720 / 1 | Correction between the TX timing marker and the conditioned RX, in 24 kS/s samples: 720 = 30 ms. The denominator must not be 0. |
| `maximum_input_age_ms` | 100 | Oldest RX input the worker still accepts. Must be > 0. |
| `stall_ms` | 5000 | Worker stall timeout. Must be > 0. |
| `hang_frames` | 51 | Hang frames sent after a call terminator (0–60). |

### 9.3 `[channel.dmr.network]`: MMDVM homebrew link

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `profile` | required | brandmeister | `brandmeister`, `dmrplus`, `tgif`, `freedmr` or `custom` (a master of your own). All speak the MMDVM homebrew protocol; only the BrandMeister profile adds the checks on `version`/`software` below. The Settings page offers the masters of the chosen network from the downloaded list (see `[dashboard.host_lists]`). |
| `endpoint` | required | 2262.master.brandmeister.network:62031 | Master `host:port`, resolved when the channel starts. |
| `password` | required | set | Login password (1–256 bytes). |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62002 | Local UDP address. Port 0 lets the system choose. |
| `callsign` | required | N0CALL | 1–8 characters. |
| `id` | `[channel.dmr] id` | 123456701 | Network ID. The effective ID must be > 1000. |
| `name` | from the host | — | Display name on the dashboard. |
| `retry_ms` | 5000 | 1000 | Login retry interval. Must be > 0. |
| `timeout_ms` | 15000 | 60000 | Session timeout. Must be greater than `retry_ms`. |

### 9.4 `[channel.dmr.network.runtime]`: station data and session policy

Every key is required except `rf_inactivity_timeout_ms`.

| Key | Station | Meaning and rules |
|---|---|---|
| `power`, `latitude`, `longitude`, `height` | 0, 0.0, 0.0, 0 | Station data sent at login: power 0–99, latitude ±90, longitude ±180, height 0–999. |
| `location` | "" | Up to 20 characters (longer text is cut). |
| `description` | "Nexus-BS 2.0 hotspot" | Up to 19 characters (longer text is cut). |
| `url` | "" | Up to 124 characters. |
| `version` | 20260916_Nexus | With BrandMeister it must be `YYYYMMDD_…`. |
| `software` | MMDVM_Nexus | With BrandMeister it must start with `MMDVM`, otherwise the master refuses the login. |
| `options` | "" | Options string sent to the master, e.g. `TS1=1;TS2=1`. |
| `retention_ms` | 250 | Network voice retention. Must be > 0. |
| `rf_inactivity_timeout_ms` | 5000 | RF inactivity timeout. Omit it, or set a value > 0. |
| `embedded_lc_only` | true | true = send only embedded link control on network voice; false = keep the incoming embedded data. |

## 10. P25

### 10.1 `[channel.p25]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `nac` | required | 0x293 | Network Access Code, 0x000–0xFFF. Write it in hex as programmed in the radios. The dashboard shows it in decimal (659). |

### 10.2 `[channel.p25.worker]`: native modem (every key required)

| Key | Station | Meaning and rules |
|---|---|---|
| `symbol_deviation`, `tx_level_q15`, `tx_peak` | 10.0, −13056, 0.5 | As for DMR (§9.2). |
| `duplex` | true | Duplex transmitter. `tx_hang` only works in duplex. |
| `tx_delay` | 0 | Original MMDVM units: 500 ms + value × 10 ms, capped at 1 s. |
| `tx_hang` | 0 | Seconds of hang after a call. |
| `network_status` | inbound_outbound | Status bits sent on the air: `inbound_busy`, `inbound_idle` or `inbound_outbound`. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | As for DMR. |
| `assembly_ms` | 500 | Window for assembling reflector voice frames (LDUs). |
| `retention_ms` | 1000 | How long a complete LDU is held for transmission. Must be less than `reflector.timeout_ms`. |
| `watchdog_ms` | 2000 | A call ends after this long without traffic. |
| `call_limit_ms` | 180000 | Absolute call length limit. |

### 10.3 `[channel.p25.reflector]`

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `endpoint` | required | p25.tetralink.ro:41000 | Reflector `host:port`. |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62005 | Local UDP address. |
| `callsign` | required | N0CALL | 1–10 characters, uppercase `A-Z 0-9 / -` only. |
| `interval_ms` | 5000 | 5000 | Poll interval. |
| `timeout_ms` | 15000 | 15000 | Link timeout. Must be greater than `interval_ms`. |
| `talkgroup` | none | — | Optional fixed route: only RF group calls to this talkgroup go to the reflector, and incoming traffic is rewritten to it. Absent means transparent. 0 is invalid. |
| `name` | from the host | — | Display name. |
| `password` | — | — | Not supported by the P25 reflector protocol: setting it is an error. |
| `auto_select` | false | — | Select the reflector from the talkgroup dialled on RF, as P25Gateway does. A group call to another talkgroup found in `hosts` or the downloaded reflector list links that reflector between calls: the over that selects it stays local, the next one goes out. Requires `talkgroup` (the default it returns to). |
| `revert_secs` | 600 | — | Seconds without traffic before returning to the configured `endpoint`/`talkgroup`. 0 stays on the last selected reflector. |
| `[[…reflector.hosts]]` | none | — | Your own reflector entries, `talkgroup` (1–65535), `endpoint` (`host:port`) and optional `name`; they win over the downloaded list. Up to 256. |

Example with automatic selection:

```toml
[channel.p25.reflector]
endpoint = "p25.tetralink.ro:41000"
talkgroup = 226          # default reflector's talkgroup
callsign = "N0CALL"
auto_select = true       # dial another TG on RF to change reflector
revert_secs = 600

[[channel.p25.reflector.hosts]]
talkgroup = 10200
endpoint = "p25.example.org:41000"
name = "North America"
```

## 11. Analog FM

The FM channel is the original MMDVM FM repeater running on the SDR, bridged to a network either through svxlink
(RoLink) or through a legacy 8 kHz USRP link. The RF channel is 25 kHz.

### 11.1 `[channel.fm]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `callsign` | required | N0CALL | Station callsign (1–23 printable characters), used in the network metadata. The CW identification text is `keyers[0].text`. |

### 11.2 `[channel.fm.modem]`: MMDVM FM repeater (every key required)

| Key | Station | Meaning and rules |
|---|---|---|
| `mode` | simplex | `link`, `simplex` or `duplex`. |
| `access` | carrier_and_ctcss | How the input opens: `carrier`, `ctcss_delayed`, `carrier_and_ctcss` or `ctcss_latch`. |
| `external_enabled` | true | Enable the network audio path. |
| `rx_level` | 200 | RF input scale, 256 / `rx_level`. Must not be 0. At 200 there is headroom; 128 clipped strong stations. |
| `tx_level` | 128 | Output scale. The svxlink `tx_ceiling` units assume 128. |
| `rf_boost` / `external_boost` | 1 / 1 | Gain multipliers when relaying RF or network audio. |
| `ctcss_frequency` | 103 | CTCSS tone as integer Hz from the MMDVM table (103 = 103.5 Hz). Valid codes: 67 69 71 74 77 79 82 85 88 91 94 97 100 103 107 110 114 118 123 127 131 136 141 146 151 156 159 162 165 167 171 173 177 179 183 186 189 192 196 199 203 206 210 218 225 229 233 241 250 254. |
| `ctcss_high` / `ctcss_low` | 150 / 100 | CTCSS detector thresholds, in original MMDVM units. |
| `ctcss_level` | 16 | CTCSS TX tone level: about 400 Hz deviation on this station. |
| `squelch_high` / `squelch_low` | 111 / 105 | Carrier squelch thresholds on the RSSI metric. The squelch opens or closes after 4 consecutive readings beyond a threshold. With `cos_invert = true`, it opens at or below `squelch_low` and closes at or above `squelch_high`. |
| `cos_invert` | true | Inverts the comparison. On this board the metric measures noise, so a strong signal gives a *low* value. Measured idle floor: 115.3. |
| `max_deviation` | 0 | Over-deviation blanking threshold × 128; 0 turns it off. It replaces excessive audio with a bleep and silence; it is **not** a limiter. It proves nothing about the RF deviation. |
| `timeout_level` | 20 | Level of the timeout tone and the blanking bleep. |
| `callsign_at_start` / `callsign_at_end` / `callsign_at_latch` | false | When to send the CW identification. |

**`[channel.fm.modem.timers]`**: all keys required, in milliseconds; 0 disables a timer.

| Key | Station | Meaning |
|---|---|---|
| `callsign` | 600000 | CW identification interval (10 min). |
| `timeout` | 180000 | Transmit timeout (3 min). |
| `holdoff` | 0 | Hold-off timer. |
| `kerchunk` | 250 | Kerchunk timer. |
| `ack_min` / `ack_delay` | 1000 / 500 | Acknowledgement timers. |
| `hang` | 300 | Repeater hang time. |

The exact MMDVM semantics of `holdoff`, `kerchunk` and the acknowledgement timers are those of the original MMDVM FM
controller. This code base does not document them further.

**`[[channel.fm.modem.keyers]]`**: exactly three blocks, in this order: callsign, RF acknowledgement, network
acknowledgement.

| Key | Station | Meaning |
|---|---|---|
| `text` | N0CALL / K / R | CW text, up to 255 characters; unknown characters are skipped. |
| `speed_wpm` | 20 / 18 / 22 | Speed, must not be 0. |
| `frequency_hz` | 1000 / 900 / 1100 | Tone frequency, 1–24000. |
| `high_level` / `low_level` | 0 / 0 | Keyer tone levels in original MMDVM units. The code base does not document them precisely. |

### 11.3 `[channel.fm.worker]`: signal path (every key required except `squelch`)

| Key | Station | Meaning |
|---|---|---|
| `symbol_deviation` | 10.0 | FM modulation scaling. Must be > 0. |
| `rx_dc_block` | true | DC blocker before demodulation. |
| `power_calibration` | 0 | Offset added to the RSSI dB reading. |
| `tx_peak` | 0.5 | Peak baseband envelope, 0 < x ≤ 0.8. |
| `host_tx_gain` | 1.0 | RF → network audio gain. Must be > 0. |
| `host_rx_gain` | 9.0 | Network → RF audio gain. Must be > 0. On this station svxlink sends flat, unlimited audio and nexus2 does the processing. |
| `pre_emphasis` / `de_emphasis` | false / false | Host-side filters. Off here because the svxlink options below do that work. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | As for DMR. |

**`[channel.fm.worker.squelch]`** (optional; every key has a default):

| Key | Default | Station | Meaning |
|---|---|---|---|
| `mode` | fixed | fixed | `fixed`: the modem thresholds above decide. `shadow`: measure and report only. `adaptive`: a noise-floor tracker decides. `adaptive` requires `squelch_high = 1`, `squelch_low = 0` and `cos_invert = false`; that is checked at start, not by `doctor`. |
| `open_db` / `close_db` | 10 / 5 | — | Signal-to-noise ratio needed to open and to stay open (0–60; close < open). Used by `adaptive`. |
| `quiet_open_db` / `quiet_close_db` | 6 / 3 | — | Quieting needed to open and to stay open (0–60; close < open). |

### 11.4 `[channel.fm.svxlink]`: bridge to svxlink / RoLink

This bridge sends raw 16 kHz PCM over UDP, with squelch and PTT over PTYs. It is exclusive with `[channel.fm.usrp]`:
configure exactly one.

| Key | Default | Station | Meaning and rules |
|---|---|---|---|
| `rx_audio` | required | 127.0.0.1:40100 | Where RF audio is sent. Matches svxlink `[Rx] AUDIO_DEV=udp:…`. |
| `tx_audio_bind` | required | 127.0.0.1:40101 | Local socket for network audio from svxlink `[Tx] AUDIO_DEV=udp:…`. Must differ from `rx_audio`. |
| `ptt_pty` | required | …/var/run/ptt | svxlink `[Tx] PTT_PTY`. |
| `squelch_pty` | required | …/var/run/sql | svxlink `[Rx] PTY_PATH` (`SQL_DET=PTY`). Must differ from `ptt_pty`. |
| `talker_file` | none | …/var/run/talker | File where svxlink writes the current talker, shown on the dashboard. |
| `tg_file` | none | …/var/run/tg | File holding the selected talkgroup. |
| `conf` | none | …/svxlink.conf | svxlink's own config, read once at start for the network name, callsign and talkgroups shown. |
| `network` | from the reflector host | RoLink | Display name. |
| `jitter_ms` | 75 | 75 | Inbound audio prebuffer (0–500). |

**Receive processing (RF → network).** Every stage is off by default; the station uses the recommended set.

| Key | Default | Station | Meaning |
|---|---|---|---|
| `rx_de_emphasis_hz` | 0 (off) | 300.0 | De-emphasis corner, 0 or 50–2000 Hz. |
| `rx_gate_depth_db` | 0 (off) | 30.0 | Depth of the noise-floor-tracking downward expander (0–60). The keys below only apply when this is > 0. |
| `rx_gate_open_db` / `rx_gate_close_db` | 10 / 5 | — | Level above the floor that counts as speech (3–40) and that holds it (1…open). |
| `rx_gate_hold_ms` / `rx_gate_attack_ms` / `rx_gate_release_ms` | 120 / 5 / 60 | — | Gate timing (hold ≤ 2000, attack 1–200, release 5–2000). |
| `rx_floor_rise_db_per_s` | 5.0 | — | How fast the floor estimate may rise (0.1–60). |
| `rx_level_target_dbfs` | off | −21.0 | Leveler target, rms of speech (−40…−6, below the peak ceiling). The next keys apply only when set. |
| `rx_level_initial_gain_db` / `rx_level_min_gain_db` / `rx_level_max_gain_db` | 20 / −10 / 30 | — | Leveler gains (min −20…max, max 0–40). |
| `rx_level_rise_db_per_s` / `rx_level_fall_db_per_s` | 8 / 30 | — | Leveler speed (0.5–60 / 1–120). |
| `rx_peak_ceiling_dbfs` | off | −6.0 | Look-ahead limiter ceiling (−20…−0.5). |

**Transmit processing (network → RF):**

| Key | Default | Station | Meaning |
|---|---|---|---|
| `tx_pre_emphasis` | false | true | nexus2 applies pre-emphasis (+6 dB/octave above 300 Hz, unity at 1 kHz). Set svxlink `[Tx1] PREEMPHASIS=0` with it. |
| `tx_limiter` | false | true | Look-ahead peak limiter at `tx_ceiling`. Set svxlink `LIMITER_THRESH=0` with it. |
| `tx_ceiling` | 2048 | 2200 | Limiter ceiling (256–3072). 1 unit ≈ 1.9 Hz deviation at `tx_level` 128, so 2200 ≈ 4.2 kHz. |

### 11.5 `[channel.fm.usrp]`: legacy USRP link (alternative to svxlink)

All keys are required. The link carries legacy 8 kHz audio.

| Key | Meaning |
|---|---|
| `bind` | Local UDP `ip:port`. It must not share a nonzero port with an earlier FM channel. |
| `peer` | Remote unicast `ip:port`, same address family as `bind` and different from it. |
| `loss_timeout_ms` | After this long without packets, the network voice is released. Must be > 0. |

### 11.6 `[callsigns]`: identities and MDC1200

This table is used by the svxlink FM bridge only. It is loaded at start and rebuilt with `nexus2 callsigns`.

| Key | Default | Station | Meaning |
|---|---|---|---|
| `enabled` | true | true | Use the identities at all. |
| `database` | none | /opt/nexus-bs2/callsigns.csv | `callsign,mdc_id` file. It must be readable at start, otherwise the start fails. |
| `source` | none | RoLink node API | Operator list used when rebuilding: a URL or a local file. |
| `selection` | active | active | `active` = operators heard within `active_days`; `all` = every operator ever listed. |
| `active_days` | 30 | 60 | Look-back window (1–3650). |
| `mdc_encode` | false | true | Send the network talker's MDC1200 ID ahead of network audio on analog FM. Outward only: MDC heard on RF is never trusted. |
| `unknown` | RoLink | ROLINK | Label announced for a talker missing from the database. Empty sends nothing. |

## 12. Deferred modes

`dstar`, `ysf`, `nxdn` and `pocsag` channels parse and appear in `plan`/`doctor`, but a live start refuses them while
they are enabled. Their tables accept `tx_delay` (all four), `low_deviation` and `hang_seconds` (YSF, default 4) and
`hang_seconds` (NXDN, default 5). They exist for planning only.

## 13. `[dashboard]`, `[network]`, `[touch]`

### `[dashboard]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `listen` | 127.0.0.1:8872 | 0.0.0.0:8080 | HTTP and WebSocket address. A non-loopback address requires `auth.enabled = true`. |
| `id_database_url` | https://radioid.net/static/user.csv | — | Source of the callsign/name/country database (https only). Refreshed when older than 7 days. |
| `id_database_dir` | `<config dir>/ids` | — | Where that database is stored. |
| `host_lists.dmr_url` | https://www.pistar.uk/downloads/DMR_Hosts.txt | — | DMR master list (Pi-Star format) for the Settings page. |
| `host_lists.p25_url` | https://refcheck.radio/api/hostfile-gate/fetch/p25/ | — | P25 reflector list (P25Hosts.txt format): the DVRef registry, served by RefCheck.Radio. Also used by P25 `auto_select`. |
| `host_lists.p25_token` | none | — | This station's personal RefCheck.Radio token. Get it with **Settings → Station → DVRef P25 reflector list → Get token**: after you accept RefCheck.Radio's terms there, the station asks for a token for its own callsign (taken from the P25 reflector, DMR network or FM callsign, suffix such as `-1` dropped), stores it here and downloads the list. **Revoke token** gives it back. Without it the P25 list is not downloaded (manual `hosts` still work). Never shown by the dashboard; removed from Share exports. |
| `host_lists.refresh_hours` | 24 | — | Download the DMR list again after this many hours (1–720). The cached copies live next to the ID database. |
| `host_lists.p25_refresh_hours` | 168 | — | The same for the P25 list: weekly by default. RefCheck.Radio allows at most one request per hour, which the station enforces whatever this says. |
| `auth.enabled` | true | true | HTTP Basic authentication on every page, API route and WebSocket. |
| `auth.username` | admin | set | 1–64 printable characters, no `:`. |
| `auth.password` | nexus | set | 1–256 characters. **Change the default.** |

### `[network]`

| Key | Default | Station | Meaning |
|---|---|---|---|
| `connect_timeout_secs` | 10 | 10 | Validated (1–300) but currently not used by any link. |
| `brew` | none | set | TETRA core link, §8.5. |

### `[touch]`: Nexus-BS Touch panel

This table is read only by the panel, which re-reads it within 5 s of a change.

| Key | Default | Meaning |
|---|---|---|
| `backlight` | 100 | % backlight in use (1–100). |
| `screen_timeout_secs` | 300 | Idle seconds before the screensaver; 0 = always on (maximum 86400). |
| `screensaver` | splash | `splash` = the boot logo, static and dimmed; `blank` = black with the backlight off. |
| `screensaver_dim` | 30 | % brightness of the dimmed logo. |
| `screensaver_backlight` | 25 | % backlight while the logo shows. |
| `wake_on_touch` | true | A touch wakes the screen. That touch is consumed and never presses a control. |
| `wake_on_traffic` | true | A new call or last-heard entry wakes the screen. |

With a timeout set, at least one of the two wake options must be on. Backlight control needs
`/sys/class/backlight` and write permission; without them the panel only dims the image.

## 14. Command line

`--config <file>` defaults to `nexus2.toml`.

| Command | Purpose |
|---|---|
| `nexus2 plan` | Show the frequency plan without hardware. |
| `nexus2 doctor [--live]` | Validate everything that can be checked offline. `--live` adds the live-start requirements (§6). |
| `nexus2 plan-risks --occupied-half-bandwidth-hz N --rx-bandwidth-hz N --tx-bandwidth-hz N [--sample-rate N]` | Offline DC, image and intermodulation geometry of the plan. |
| `nexus2 init [--defaults]` | Create a new file (TX inhibited). Refuses to overwrite an existing one. |
| `nexus2 callsigns --out FILE [...]` | Rebuild the callsign/MDC1200 database from `[callsigns]`. |
| `nexus2 run --live [--allow-rf-tx]` | The station itself (the systemd service). |
| `nexus2 manage ...` | The HTTPS management API (separate service). |

## 15. Recipes

**Move a carrier.** Change its `offset_hz` by a multiple of 25 kHz. Keep 50 kHz from its neighbours and stay inside
±(sample rate / 2 − 20 kHz). Run `nexus2 plan`, then `doctor --live` (TETRA must keep a standard encoding), apply and
restart.

**Take a mode off the air for good.** Set `enabled = false` on its `[[channel]]`. It keeps its frequency slot. For
DMR, P25 or FM a temporary off can be done live from the dashboard, and it lasts until the next restart.

**Enable the TETRA packet-data carrier.** Set `enabled = true` on `tetra-data`, keeping its identity equal to the
primary's, and restart. The primary keeps its `[channel.tetra.packet_data]`.

**Adjust a mode's transmit level.** Change that mode's `tx_peak`: `[channel.tetra.worker]`, `[channel.dmr.worker]`,
`[channel.p25.worker]` or `[channel.fm.worker]`. Keep it at 0.8 or below. The total of all carriers shares one DAC, so
raising one mode raises the composite peak. `sdr.tx_gain_db` and the element gains move **all** modes together.

**Measure receive levels.** Set `sdr.io_diagnostics = true`, restart, collect the `RX_LEVEL` journal lines, then set
it back to `false`.

**Change the dashboard password.** Set `dashboard.auth.password` and restart the radio service. The touch panel
logs in to the dashboard with the credentials from this same file, but reads them only once when it starts, so
restart `nexus-panel.service` as well.

## 16. Known limitations and open points

These come from the code as it stands; they are reported so that nobody relies on them:

- `network.connect_timeout_secs` is validated but no link uses it.
- The TETRA `tx_peak` limit of 0.8 documented in the code is not enforced; only 0–1.0 is, when TX is enabled.
- `packet_data.max_slots` outside 1–4 is clamped silently.
- `sdr.driver` does not select the device (`uri` does). A wrong driver name only changes the SX1255-specific
  behaviour.
- `nexus2 callsigns` quietly falls back to defaults if the file does not load, and may probe the SDR when `[sdr]`
  uses auto.
- Not documented in this code base: the exact units of DMR `receiver_delay`; what exactly triggers `stall_ms`; the FM
  keyer `high_level`/`low_level` and CTCSS threshold/level scales (original MMDVM byte units); the longest CW text that
  fits.
- Per-channel settings reload without a restart exists in the code but has no production caller. Only the DMR, P25
  and FM on/off toggle is live.
