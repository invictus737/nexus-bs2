# Konfigurationshandbuch für Nexus-BS 2.0 (`nexus2.toml`)

Englische Version: [Manual_conf.md](Manual_conf.md)

Rumänische Version: [Manual_conf.ro.md](Manual_conf.ro.md)

Dieses Handbuch erklärt jeden Schlüssel der Stationskonfigurationsdatei, wie die Datei geprüft wird und wie
Änderungen die laufende Station erreichen. Es wurde anhand des Quellcodes geschrieben (Commit `d7fdec7` und später),
nicht aus dem Gedächtnis. Wo der Code ein Detail nicht festlegt, sagt das Handbuch dies, statt zu raten.

Die Konfiguration der in Betrieb befindlichen Station ist die Referenz für empfohlene Werte. Das einzige mit Nexus-BS 2.0
ausgelieferte Beispiel, `examples/nexus2.example.toml`, verwendet dieselben Einstellungen mit generischen
Identitäten (`N0CALL`, `CHANGE_ME`), dem Frequenzplan des TETRA-BlueStation-Beispiels und gesperrtem TX. In den folgenden Tabellen gilt:

- **Standard** ist der Wert, den der Code verwendet, wenn der Schlüssel fehlt. „erforderlich“ bedeutet, dass es keinen Standardwert gibt.
- **Station** ist der Wert in der Datei der in Betrieb befindlichen Station. Ein Strich bedeutet, dass der Schlüssel dort fehlt.

Inhalt:
1. [Wie die Datei gelesen wird](#1-wie-die-datei-gelesen-wird)
2. [Eine Änderung anwenden](#2-eine-änderung-anwenden)
3. [Frequenzen und der Kanalplan](#3-frequenzen-und-der-kanalplan)
4. [Sendeerlaubnis und der RF-Schalter](#4-sendeerlaubnis-und-der-rf-schalter)
5. [`[sdr]`](#5-sdr-das-gemeinsame-funkgerät)
6. [`[[channel]]`](#6-channel-gemeinsame-schlüssel)
7. [`[channel.access]`](#7-channelaccess-zulassungsrichtlinie)
8. [TETRA](#8-tetra)
9. [DMR](#9-dmr)
10. [P25](#10-p25)
11. [Analog-FM](#11-analog-fm)
12. [Zurückgestellte Betriebsarten](#12-zurückgestellte-betriebsarten)
13. [`[dashboard]`, `[network]`, `[touch]`](#13-dashboard-network-touch)
14. [Kommandozeile](#14-kommandozeile)
15. [Rezepte](#15-rezepte)
16. [Bekannte Einschränkungen und offene Punkte](#16-bekannte-einschränkungen-und-offene-punkte)

---

## 1. Wie die Datei gelesen wird

- **Format.** Die Datei ist TOML 1.0. Innerhalb einer Tabelle ist die Reihenfolge der Schlüssel beliebig. Ein
  Tabellenkopf wie `[network.brew]` darf überall in der Datei stehen, sogar nach den `[[channel]]`-Einträgen. Die
  Stationsdatei nutzt das, um jede Netzanbindung zusammen mit ihrer Betriebsart zu gruppieren.
- **Strikt.** Jede Tabelle weist unbekannte Schlüssel zurück. Ein Tippfehler ist ein harter Fehler, zum Beispiel
  `config: unknown field `brightnes` near line 70`. Fehlermeldungen nennen den Schlüsselpfad und die Zeilennummer,
  niemals den Wert, damit keine Geheimnisse in Logs gelangen.
- **Erforderliche Tabellen.** `[sdr]`, mindestens ein `[[channel]]`, `[dashboard]` und `[network]` sind erforderlich.
  `[dashboard]` und `[network]` dürfen leer sein. `[touch]` und `[callsigns]` sind optional.
- **Wertformen.**
  - Eine Ganzzahl wird akzeptiert, wo eine Dezimalzahl erwartet wird (`rx_gain_db = 30`).
  - Hex-, Oktal- und Binärliterale funktionieren für jede Ganzzahl (`nac = 0x293`), ebenso Unterstriche (`offset_hz = -50_000`).
  - Negative Zahlen müssen in Dezimalschreibweise angegeben werden.
- **Einheiten.** Frequenzen sind in Hz. Zeiten sind in ms, sofern der Schlüsselname nicht auf `_secs` endet.
- **Geheimnisse.** Geheimnisse sind einfache Zeichenketten: `dashboard.auth.password`, `network.brew.password`,
  `channel.dmr.network.password` und `sdr.uri`. Sie werden in Debug-Ausgaben und im geschwärzten Export verborgen.
  Halten Sie den Dateimodus auf 0600. Das Dashboard und die Management-API schreiben die Datei beide so.

## 2. Eine Änderung anwenden

**Der Funkprozess liest die Datei einmal, beim Start.** Das Bearbeiten der Datei ändert nichts, bis der Dienst neu
startet. Es gibt zwei Ausnahmen:

| Was | Wird wirksam |
|---|---|
| `[touch]` | Das Panel liest sie innerhalb von etwa 5 s neu ein. Ein Wert außerhalb des zulässigen Bereichs fällt auf seinen Standardwert zurück. |
| Aktivieren oder Deaktivieren eines **DMR-, P25- oder FM**-Kanals über den Umschalter im Dashboard oder Panel | Sofort, ohne Neustart. Der Zustand wird **nur im Speicher** gehalten und nicht in die Datei geschrieben: nach einem Neustart des Dienstes gilt wieder der `enabled`-Wert der Datei. TETRA hat keinen Live-Umschalter. |

Alles andere erfordert einen Neustart. `[callsigns]` wird ebenfalls nur beim Start geladen.

**Vor dem Anwenden prüfen:**

```sh
nexus2 --config nexus2.toml plan         # frequency plan, no hardware
nexus2 --config nexus2.toml doctor       # full static validation
nexus2 --config nexus2.toml doctor --live   # plus the checks a live start makes
```

`doctor` öffnet niemals das SDR oder das Netz. Es löst kein DNS auf, bindet keine Sockets und liest die
`[callsigns]`-Datenbank nicht; ein fehlerfreies `doctor` garantiert daher nicht, dass die Netzanbindungen zustande
kommen.

**Auf der Station** (Management-API von einem Arbeitsplatzrechner aus; siehe MANAGEMENT-API.md):

```sh
A="python3 scripts/nexus2-admin.py --url https://<station-ip>:9443 --connect-ip <station-ip>"
$A config > current.json                 # contains secrets: keep private
$A validate candidate.toml               # validate with the station's own checks
$A set-config candidate.toml --if-match <config_sha256 from status>
```

- `set-config` tauscht die Datei aus, behält eine Sicherungskopie, startet den Funkdienst neu und stellt den
  vorherigen Stand wieder her, wenn der Prozess nicht fehlerfrei weiterläuft.
- Die Hash-Vorbedingung verweigert das Überschreiben einer Datei, die sich zwischenzeitlich geändert hat.
- Der Neustart unterbricht kurz **jede Betriebsart**.
- Der Manager validiert mit seinem eigenen Binary. Nach einem Upgrade der Funksoftware, das neue Schlüssel hinzufügt,
  muss auch der Manager aktualisiert werden, sonst weist er sie zurück (HTTP 422).
- Solange die Management-API installiert ist (Markierungsdatei `/opt/nexus-bs2/.management-enabled`), verweigern der
  eigene Konfigurationseditor und das Einstellungsformular des Dashboards das Speichern und antworten mit
  `use_management_api`.

## 3. Frequenzen und der Kanalplan

Ein SDR trägt alle Carrier. Jeder Kanal ist ein Offset gegenüber den beiden Local Oscillators (LO):

```
RX frequency (uplink, terminals -> station)   = sdr.rx_lo_hz + offset_hz
TX frequency (downlink, station -> terminals) = sdr.tx_lo_hz + offset_hz
```

Es gibt ein `offset_hz` pro Kanal, daher hat jeder Carrier denselben Duplex Split: `tx_lo_hz - rx_lo_hz`.

**Planregeln.** Diese werden für **alle** Kanäle geprüft, deaktivierte eingeschlossen:

| Regel | Fehler bei Verstoß |
|---|---|
| 1 bis 7 Kanäle insgesamt | Kanalanzahl |
| `offset_hz` ist ein exaktes Vielfaches von 25 000 | `channel[i].offset_hz: must be an exact multiple of 25000` |
| Je zwei Kanäle mindestens 50 000 Hz voneinander entfernt (eine leere 25-kHz-Zelle zwischen Carriern) | Planfehler |
| `abs(offset_hz) + 20 000 < sample_rate / 2` | "insufficient Nyquist/filter margin" |
| SX1255-Boards (`sx`, `mucell`): kein **aktivierter** Kanal bei `offset_hz = 0`, weil er auf DC läge | "carrier would sit on DC; shift rx_lo_hz/tx_lo_hz by a 25 kHz multiple instead" |

- Das erforderliche analoge Passband ist `2 × (max |offset_hz| + 20 000)`. Für die Station sind das
  2 × (175 000 + 20 000) = 390 kHz.
- **Sample Rate.** Wenn `sdr.sample_rate` fehlt oder 0 ist, fragt die Station das Funkgerät, welche Raten es
  unterstützt. Sie behält jene, die höchstens 1 MHz betragen, ein Vielfaches von 50 kHz sind und breiter als das
  erforderliche Passband sind. Dann nimmt sie die kleinste mit mindestens dem doppelten Passband, andernfalls die
  breiteste verwendbare. Die Station landet bei 600 kS/s.
- **TETRA-Frequenzkodierung.** Ein TETRA-Carrier muss außerdem ein Standard-TETRA-Kanal sein:
  - Die TX-Frequenz muss auf dem 25-kHz-Raster liegen, optional verschoben um +6.25, −6.25 oder +12.5 kHz.
  - Das RX–TX-Spacing muss eines der Standard-Duplex-Spacings des Bandes sein. Im 400-MHz-Band sind dies 10, 7, 8,
    5 oder 9.5 MHz. Die Station verwendet 7 MHz.
  - Andernfalls lautet der Fehler `tetra: frequencies have no supported standard repeater encoding`.
  - `plan` prüft dies nicht; `doctor` und ein Live-Start tun es.

**Stationsplan** (LOs 431.2875 / 438.2875 MHz):

| Kanal | offset_hz | RX MHz | TX MHz | Zustand |
|---|---|---|---|---|
| tetra-data | −50 000 | 431.2375 | 438.2375 | deaktiviert |
| dmr | +25 000 | 431.3125 | 438.3125 | aktiviert |
| tetra | +75 000 | 431.3625 | 438.3625 | aktiviert |
| p25 | +125 000 | 431.4125 | 438.4125 | aktiviert |
| analog (FM) | +175 000 | 431.4625 | 438.4625 | aktiviert |

Um die gesamte Station zu verschieben, ändern Sie beide LOs um denselben Betrag. Um einen Carrier zu verschieben,
ändern Sie dessen `offset_hz`, halten Sie die Rasterregeln ein und prüfen Sie mit `nexus2 plan`.

## 4. Sendeerlaubnis und der RF-Schalter

Drei voneinander unabhängige Dinge entscheiden, ob die Station sendet:

1. **`sdr.tx_inhibited`** (Konfiguration, Code-Standardwert `true`). Wenn true, wird überhaupt kein TX-Pfad
   vorbereitet. Die Stationsdatei setzt `false`. `nexus2 init` schreibt immer `true`.
2. **`--allow-rf-tx`** (Kommandozeile von `run --live`). Dies ist nur die Erlaubnis für den Prozess. Eine Datei mit
   `tx_inhibited = false` wird ohne diese Option beim Start zurückgewiesen. Die Service-Unit übergibt sie.
3. **Der globale RF-Schalter** (RF KILL / RESTORE RF im Dashboard und Panel). Dies ist **kein Konfigurationsschlüssel**.
   - Sein Zustand wird in `~/.local/state/nexus2/rf-<hash of the config path>` gespeichert und übersteht daher
     Neustarts. OFF stoppt RX und TX aller Betriebsarten.
   - Nur ein RF KILL durch den Operator oder ein fataler Fehler beim sauberen Stoppen des Funkgeräts speichert OFF.
   - Ein fehlgeschlagener Start behält ON bei und versucht es selbstständig erneut, nach 2 s und dann mit wachsender
     Pause bis zu 60 s. Ein Beispiel ist ein Netzwerkname, der sich während des Bootens noch nicht auflösen lässt.
   - Dies ist eine Softwaresteuerung, keine Hardware-Verriegelung.

## 5. `[sdr]`: das gemeinsame Funkgerät

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `driver` | `"auto"` | — | SoapySDR-Treibername, oder `auto`/leer, um das beste gefundene Funkgerät zu verwenden. Er wird **nicht** zum Öffnen des Geräts verwendet: das tut `uri`. Er teilt der Station nur mit, ob das Board ein SX1255 ist (`sx`, `mucell`). Das beeinflusst die DC-Regel, den Standardwert von `tx_lead_quanta` und eine Warnung zum aufgesteckten HAT. Automatische Bevorzugungsreihenfolge: SX1255-Boards (`sx` vor `mucell`), Lime, USRP B2xx, andere USRP, Pluto, alles andere. |
| `uri` | `"auto"` | — | SoapySDR-Geräteargumente (1–256 Zeichen), oder `auto`, um sie vom gefundenen Funkgerät zu übernehmen. |
| `sample_rate` | `0` (auto) | — | Komplexe Sample Rate des gesamten Geräts. Muss ein positives Vielfaches von 50 000 und höchstens 1 000 000 sein. Siehe §3. |
| `rx_lo_hz` | erforderlich | 431287500 | RX Local Oscillator. |
| `tx_lo_hz` | erforderlich | 438287500 | TX Local Oscillator. |
| `rx_gain_db` | erforderlich | 30.0 | Gesamte RX-Verstärkung (−100…100), angewendet vor den Element-Verstärkungen. |
| `tx_gain_db` | erforderlich | 0.0 | Gesamte TX-Verstärkung (−100…100). |
| `rx_gains_db.<element>` | keiner | LNA 42, PGA 16 | Verstärkungen je RX-Element, angewendet nach der Gesamtverstärkung. Die Elementnamen stammen vom Treiber. |
| `tx_gains_db.<element>` | keiner | DAC 9, MIXER 30 | Verstärkungen je TX-Element. |
| `rx_bandwidth_hz` / `tx_bandwidth_hz` | auto | — | Breite des Analogfilters. Falls gesetzt, muss sie **größer als** das erforderliche Passband sein (§3). Fehlt der Schlüssel, wird der Bereich des Treibers verwendet. |
| `rx_channel` / `tx_channel` | 0 | — | Hardware-Kanalindex bei Mehrkanal-Funkgeräten. |
| `rx_antenna` / `tx_antenna` | je Board | — | Antennenanschluss. Die Standardwerte sind SX1255 `RX`/`TX`, Lime USB `LNAL`/`BAND1`, andere Lime `LNAW`/`BAND2`, Pluto `A_BALANCED`/`A`, USRP `TX/RX`. |
| `rx_stream_args` / `tx_stream_args` | leer | — | Zusätzliche SoapySDR-Stream-Argumente (`key = "value"`). |
| `settings.<key>` | leer | — | Geräteweite SoapySDR-Einstellungen. SX1255-Boards erhalten `PA = "AUTO"`, sofern hier nicht anders gesetzt. |
| `ppm` | 0.0 | — | Frequenzkorrektur für beide Pfade (−100…100). Ein Wert ungleich null erfordert einen Treiber, der Korrektur unterstützt, sonst schlägt das Öffnen des Funkgeräts fehl. |
| `tx_lead_quanta` | 8 bei SX1255, sonst 3 | — | Wie viele 1-ms-Blöcke an TX-Samples dem Takt des Funkgeräts voraus vorbereitet werden (1–8). 8 ist am SX1255 gemessen; 3 ist bei USB-Funkgeräten nicht gemessen. Erhöhen Sie diesen Wert als Erstes, wenn das Log "TX late block skipped" zeigt. |
| `sx1255_rx_pll_bw_khz` | Treiber (300) | — | Loop-Bandbreite der SX1255-RX-PLL: 75, 150, 225 oder 300. Wird bei anderen Funkgeräten zurückgewiesen. |
| `sx1255_tx_pll_bw_khz` | Treiber (150) | 300 | Loop-Bandbreite der SX1255-TX-PLL. 300 ergibt das niedrigste Close-in Phase Noise auf dem Carrier. |
| `tx_inhibited` | **true** | false | Siehe §4. |
| `io_diagnostics` | false | false | Messhilfe: RX-Pegelfenster und I/O-Timing-Zähler sowie der `rf_io`-Block im Dashboard. Schreibt etwa 11 Journalzeilen pro Sekunde, daher im Normalbetrieb ausgeschaltet lassen. Verändert nichts am RF. |
| `pipeline_diagnostics` | false | — | Experimentelle Instrumentierung von Warteschlangen und Kanälen. |

## 6. `[[channel]]`: gemeinsame Schlüssel

Jeder `[[channel]]`-Block ist ein Carrier. Seine Betriebsartentabelle muss ihm folgen, zusammen mit `[channel.access]`
für DMR, P25 und FM.

| Schlüssel | Standard | Bedeutung und Regeln |
|---|---|---|
| `id` | erforderlich | Eindeutiger Name, 1–64 Zeichen aus `A-Z a-z 0-9 - _`. Das Dashboard, die Steuer-API und eine Packet-Data-`cell` beziehen sich über ihn auf Kanäle. |
| `mode` | erforderlich | `tetra`, `dmr`, `p25` oder `fm`. `dstar`, `ysf`, `nxdn` und `pocsag` werden ebenfalls geparst, aber ein Live-Start weist sie zurück, wenn sie aktiviert sind (§12). Die passende Tabelle (`[channel.tetra]`, …) ist erforderlich, und jede andere Betriebsartentabelle ist ein Fehler. |
| `enabled` | true | Administratives Ein/Aus. Ein deaktivierter Kanal zählt weiterhin für die Planregeln und die Grenze von 7 Carriern. |
| `offset_hz` | erforderlich | Offset gegenüber beiden LOs (§3). |

**Reihenfolge der `[[channel]]`-Blöcke.** Die Reihenfolge ändert keine Frequenz. Sie legt den Index fest, unter dem
ein Kanal in Fehlermeldungen (`channel[2].…`) und im Einstellungsformular des Dashboards angezeigt wird.

- Der TETRA-TX-Pegel wird vom ersten aktivierten TETRA-Kanal übernommen, der einen `[channel.tetra.worker]` hat
  (`tx_peak`). Halten Sie den primären TETRA-Kanal an erster Stelle.
- USRP-Portkonflikte werden nur gegen frühere Kanäle geprüft.
- Umsortieren, Hinzufügen oder Entfernen von Kanälen erfordert einen Neustart.

**Anforderungen beim Live-Start.** Diese gehen über das hinaus, was `doctor` ohne `--live` prüft:

- Genau ein aktivierter primärer TETRA-Kanal, damit die Station nicht ohne TETRA läuft.
- Jeder andere aktivierte TETRA-Kanal muss der Packet-Data-Carrier dieser Zelle sein.
- Ein aktivierter DMR-Kanal benötigt `worker`, `network` und `network.runtime`.
- Ein aktivierter P25-Kanal benötigt `worker` und `reflector`.
- Ein aktivierter FM-Kanal benötigt `modem`, `worker` und entweder `svxlink` oder `usrp`.
- Jeder aktivierte DMR-, P25- oder FM-Kanal benötigt `[channel.access]`.

## 7. `[channel.access]`: Zulassungsrichtlinie

Diese Tabelle entscheidet, welchen Verkehr eine Betriebsart durchlässt. Standardmäßig wird nichts zugelassen, und sie
gewährt niemals physisches TX. `mode` muss der Betriebsart des Kanals entsprechen. TETRA hat keine Zugangstabelle.

| Betriebsart | Schlüssel | Bedeutung |
|---|---|---|
| dmr | `rf_slots = [TS1, TS2]` (erforderlich) | RF-Verkehr (Sprache, Daten, CSBK) auf jedem Timeslot zulassen. |
| dmr | `network_slots = [TS1, TS2]` (erforderlich) | Netzverkehr in Richtung RF auf jedem Timeslot zulassen. |
| dmr | `protected_voice` (Standard false) | Sprache mit dem Privacy-/Verschlüsselungs-Flag (PI) zulassen. Bei false wird sie in beiden Richtungen abgewiesen. |
| p25 | `rf` (erforderlich) | RF-Calls zulassen. |
| p25 | `network` (erforderlich) | Reflector-Verkehr in Richtung RF zulassen. |
| fm | `selected` (erforderlich) | Der MMDVM-Eingang „FM mode selected“ der Repeater-Logik. Er ist nicht der Carrier-/CTCSS-Zugang. |

## 8. TETRA

### 8.1 `[channel.tetra]`

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `mcc` | erforderlich | 901 | Mobile Country Code (MCC) (0–1023). |
| `mnc` | erforderlich | 9999 | Mobile Network Code (MNC) (0–16383). |
| `location_area` | erforderlich | 2 | Basis-Location-Area (0–16383). |
| `colour_code` | erforderlich | 1 | Colour Code (0–63). |
| `rotate_location_area_on_start` | true | — | Wenn true, sendet jeder Start eine andere, von `location_area` abgeleitete LA aus, sodass eingebuchte Endgeräte sich nach einem Neustart neu registrieren. Auf false setzen, um exakt `location_area` auszusenden. |
| `implicit_group_affiliation` | true | — | Ein Call- oder Floor-Request eines Endgeräts, das nicht als dieser Group zugeordnet bekannt ist, ordnet es der Group zu, statt abgewiesen zu werden. |
| `station_name` | `"Nexus-BS 2.0"` | — | Text der periodischen Home-Mode-Display-Meldung, die auf Endgeräten angezeigt wird. Leer schaltet sie aus. |
| `timezone` | Zeitzone des Hosts | — | IANA-Zonenname (z. B. `Europe/Bucharest`) für die Uhrzeitaussendung der Zelle. Leer schaltet die Uhrzeit aus. Ein ungültiger Name lässt `doctor` fehlschlagen. |
| `role` | `"primary"` | — / `packet_data` | `packet_data` macht diesen Kanal zu einem sekundären Data Carrier (§8.4). |
| `cell` | keiner | — / `tetra` | Nur für `role = "packet_data"`: die `id` des primären Kanals. |

### 8.2 `[channel.tetra.packet_data]`: IP über TETRA (nur primärer Kanal)

Wenn vorhanden, bietet die Zelle SNDCP Packet Data an. Endgeräte erhalten Adressen aus dem Pool und erreichen die
Station über eine Linux-tun-Schnittstelle.

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `ipv4_pool` | erforderlich | `10.44.0.0/24` | CIDR mit Präfix /16…/30 und ohne gesetzte Host-Bits. Der erste Host ist das Gateway auf der tun-Schnittstelle (10.44.0.1); der Rest wird an Endgeräte vergeben. |
| `tun` | `"tetra0"` | — | Name der tun-Schnittstelle (1–15 Zeichen). |
| `mtu` | 576 | — | N-PDU-Größe und Schnittstellen-MTU (128–1500). |
| `header_compression` | true | false | RFC-1144-Headerkompression (Van Jacobson) gewähren, wenn ein Endgerät sie anfordert. Das Motorola MXP600 verwirft die Aktivierung, wenn sie gewährt wird, daher lässt die Station sie ausgeschaltet. |
| `max_slots` | 4 | — | Höchstzahl der Timeslots, die ein Endgerät auf dem Data Carrier nutzen darf. Werte außerhalb von 1–4 werden **stillschweigend begrenzt**, nicht zurückgewiesen. |
| `advertise` | true | — | „SNDCP available“ in der Systeminformation ankündigen. Wenn aus, antwortet der Dienst dennoch Endgeräten, die es versuchen. |
| `advertise_advanced_link` | true | — | „advanced link supported“ ankündigen. |

### 8.3 `[channel.tetra.worker]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `tx_peak` | 0.5 | 0.8 | Spitzenwert der Basisband-Hüllkurve des TETRA-Carriers, auf derselben Skala wie `tx_peak` bei DMR, P25 und FM. 0.5 ergibt dieselbe Spitze wie DMR; etwa 0.79 ergibt dieselbe mittlere Leistung, mit 4 dB höheren Spitzen. Halten Sie den Wert bei 0.8 oder darunter. Der Code weist nur Werte außerhalb von 0–1.0 zurück, und nur wenn TX aktiviert ist. |

### 8.4 Packet-Data-Carrier (`role = "packet_data"`)

Ein zweiter TETRA-Carrier derselben Zelle, mit Packet Data auf allen vier Timeslots und ohne Control Channel.
Endgeräte werden per Channel Allocation vom primären Carrier dorthin geschickt. Regeln:

- `cell` muss einen existierenden primären TETRA-Kanal benennen, und dieser primäre Kanal muss
  `[channel.tetra.packet_data]` haben.
- `mcc`, `mnc`, `location_area` und `colour_code` müssen denen des primären Kanals gleichen.
- Ein Packet-Data-Carrier hat keine eigene `packet_data`-Tabelle, und es gibt höchstens einen pro primärem Kanal.
- Seine Frequenzen müssen im Band des primären Kanals liegen, mit demselben Duplex Spacing.
- Wenn er deaktiviert ist (`enabled = false`), bietet die Zelle Packet Data nur auf dem primären Carrier an.

### 8.5 `[network.brew]`: TETRA-Core-Anbindung (Brew / TetraPack)

Diese Tabelle erfordert mindestens einen aktivierten TETRA-Kanal.

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `host` | erforderlich | core.tetrapack.online | Hostname oder IP des Servers. |
| `port` | 443 | 443 | Serverport. |
| `tls` | true | true | TLS verwenden (wss/https). |
| `username` / `password` | keiner | gesetzt | HTTP-Digest-Anmeldung. Entweder beide oder keiner. |
| `reconnect_delay_secs` | 15 | 15 | Pause vor dem erneuten Verbinden (1–300). |
| `jitter_initial_latency_frames` | 0 | — | Zusätzlicher anfänglicher Playout-Delay in Frames für Netz-Sprache. |
| `feature_sds_enabled` | true | — | SDS-Nachrichten zwischen lokalen und Netz-Endgeräten weiterleiten. |
| `feature_rssi_export` | false | — | RSSI an den Server melden. |
| `whitelisted_ssis` | keiner | — | Nur Netz-Calls mit diesen entfernten SSIs zulassen (bis zu 4096 Einträge, jeder höchstens 0xFFFFFF). |

Die Altschlüssel `network.brew_endpoint` und `network.brew_token` werden noch geparst, aber `doctor` und ein
Live-Start weisen sie zurück. Verwenden Sie stattdessen `[network.brew]`.

## 9. DMR

### 9.1 `[channel.dmr]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `id` | erforderlich | 1234567 | 24-Bit-Repeater-ID auf RF (1–16777215). Sie ist auch die Netz-ID, sofern `network.id` nicht gesetzt ist. |
| `colour_code` | erforderlich | 1 | Colour Code (0–15). |

### 9.2 `[channel.dmr.worker]`: natives Modem (jeder Schlüssel erforderlich)

| Schlüssel | Station | Bedeutung und Regeln |
|---|---|---|
| `symbol_deviation` | 10.0 | FM-Skalierung der 4FSK-Symbole, verwendet für die RX-Aufbereitung und TX. Muss > 0 sein. |
| `tx_level_q15` | −13056 | Vorzeichenbehaftete Q15-Verstärkung für die TX-Deviation. −13056 = −102 × 128, der Referenz-SDR-Pegel. Ein negativer Wert invertiert die Deviation. |
| `tx_peak` | 0.5 | Spitzenwert der Basisband-Hüllkurve, 0 < x ≤ 0.8. |
| `power_calibration` | 0 | Offset, der zum RSSI-dB-Messwert addiert wird. |
| `receiver_delay` | 3 | Per-Slot-Delay des Repeater-Empfängers. Die genaue Einheit ist in dieser Codebasis nicht dokumentiert; 3 beibehalten. |
| `marker_offset_numerator` / `marker_offset_denominator` | 720 / 1 | Korrektur zwischen dem TX-Timing-Marker und dem aufbereiteten RX, in Samples bei 24 kS/s: 720 = 30 ms. Der Nenner darf nicht 0 sein. |
| `maximum_input_age_ms` | 100 | Ältester RX-Input, den der Worker noch akzeptiert. Muss > 0 sein. |
| `stall_ms` | 5000 | Stall-Timeout des Workers. Muss > 0 sein. |
| `hang_frames` | 51 | Nach einem Call-Terminator gesendete Hang Frames (0–60). |

### 9.3 `[channel.dmr.network]`: MMDVM-Homebrew-Anbindung

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `profile` | erforderlich | brandmeister | `brandmeister`, `dmrplus`, `tgif`, `freedmr` oder `custom` (ein eigener Master). Alle sprechen das MMDVM-Homebrew-Protokoll; nur das BrandMeister-Profil fügt die unten beschriebenen Prüfungen zu `version`/`software` hinzu. Die Settings-Seite bietet die Master des gewählten Netzes aus der heruntergeladenen Liste an (siehe `[dashboard.host_lists]`). |
| `endpoint` | erforderlich | 2262.master.brandmeister.network:62031 | Master-`host:port`, aufgelöst beim Start des Kanals. |
| `password` | erforderlich | gesetzt | Anmeldepasswort (1–256 Bytes). |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62002 | Lokale UDP-Adresse. Port 0 lässt das System wählen. |
| `callsign` | erforderlich | N0CALL | 1–8 Zeichen. |
| `id` | `[channel.dmr] id` | 123456701 | Netz-ID. Die effektive ID muss > 1000 sein. |
| `name` | vom Host | — | Anzeigename im Dashboard. |
| `retry_ms` | 5000 | 1000 | Intervall für erneute Anmeldeversuche. Muss > 0 sein. |
| `timeout_ms` | 15000 | 60000 | Sitzungs-Timeout. Muss größer als `retry_ms` sein. |

### 9.4 `[channel.dmr.network.runtime]`: Stationsdaten und Sitzungsrichtlinie

Jeder Schlüssel ist erforderlich außer `rf_inactivity_timeout_ms`.

| Schlüssel | Station | Bedeutung und Regeln |
|---|---|---|
| `power`, `latitude`, `longitude`, `height` | 0, 0.0, 0.0, 0 | Bei der Anmeldung gesendete Stationsdaten: Leistung 0–99, Breitengrad ±90, Längengrad ±180, Höhe 0–999. |
| `location` | "" | Bis zu 20 Zeichen (längerer Text wird abgeschnitten). |
| `description` | "Nexus-BS 2.0 hotspot" | Bis zu 19 Zeichen (längerer Text wird abgeschnitten). |
| `url` | "" | Bis zu 124 Zeichen. |
| `version` | 20260916_Nexus | Bei BrandMeister muss der Wert `YYYYMMDD_…` lauten. |
| `software` | MMDVM_Nexus | Bei BrandMeister muss der Wert mit `MMDVM` beginnen, sonst verweigert der Master die Anmeldung. |
| `options` | "" | An den Master gesendete Options-Zeichenkette, z. B. `TS1=1;TS2=1`. |
| `retention_ms` | 250 | Vorhaltezeit für Netz-Sprache. Muss > 0 sein. |
| `rf_inactivity_timeout_ms` | 5000 | RF-Inaktivitäts-Timeout. Weglassen oder einen Wert > 0 setzen. |
| `embedded_lc_only` | true | true = bei Netz-Sprache nur Embedded Link Control senden; false = die eingehenden Embedded-Daten beibehalten. |

## 10. P25

### 10.1 `[channel.p25]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `nac` | erforderlich | 0x293 | Network Access Code (NAC), 0x000–0xFFF. Hexadezimal schreiben, wie in den Funkgeräten programmiert. Das Dashboard zeigt ihn dezimal an (659). |

### 10.2 `[channel.p25.worker]`: natives Modem (jeder Schlüssel erforderlich)

| Schlüssel | Station | Bedeutung und Regeln |
|---|---|---|
| `symbol_deviation`, `tx_level_q15`, `tx_peak` | 10.0, −13056, 0.5 | Wie bei DMR (§9.2). |
| `duplex` | true | Duplex-Sender. `tx_hang` funktioniert nur im Duplex-Betrieb. |
| `tx_delay` | 0 | Original-MMDVM-Einheiten: 500 ms + Wert × 10 ms, begrenzt auf 1 s. |
| `tx_hang` | 0 | Sekunden Hang Time nach einem Call. |
| `network_status` | inbound_outbound | Auf der Luftschnittstelle gesendete Statusbits: `inbound_busy`, `inbound_idle` oder `inbound_outbound`. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Wie bei DMR. |
| `assembly_ms` | 500 | Fenster zum Zusammensetzen von Reflector-Sprachframes (LDUs). |
| `retention_ms` | 1000 | Wie lange eine vollständige LDU zum Senden vorgehalten wird. Muss kleiner als `reflector.timeout_ms` sein. |
| `watchdog_ms` | 2000 | Ein Call endet nach dieser Zeit ohne Verkehr. |
| `call_limit_ms` | 180000 | Absolute Begrenzung der Call-Dauer. |

### 10.3 `[channel.p25.reflector]`

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `endpoint` | erforderlich | p25.tetralink.ro:41000 | Reflector-`host:port`. |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62005 | Lokale UDP-Adresse. |
| `callsign` | erforderlich | N0CALL | 1–10 Zeichen, nur `A-Z 0-9 / -` in Großschreibung. |
| `interval_ms` | 5000 | 5000 | Poll-Intervall. |
| `timeout_ms` | 15000 | 15000 | Link-Timeout. Muss größer als `interval_ms` sein. |
| `talkgroup` | keiner | — | Optionale feste Route: nur RF-Group-Calls an diese Talkgroup gehen zum Reflector, und eingehender Verkehr wird auf sie umgeschrieben. Fehlt der Schlüssel, ist der Betrieb transparent. 0 ist ungültig. |
| `name` | vom Host | — | Anzeigename. |
| `password` | — | — | Vom P25-Reflector-Protokoll nicht unterstützt: das Setzen ist ein Fehler. |
| `auto_select` | false | — | Wählt das Reflector anhand der auf RF gewählten Talkgroup, wie bei P25Gateway. Ein Gruppenruf an eine andere Talkgroup, die in `hosts` oder in der heruntergeladenen Reflector-Liste gefunden wird, verbindet dieses Reflector zwischen den Calls: der Over, der es auswählt, bleibt lokal, der nächste geht raus. Erfordert `talkgroup` (den Standard, zu dem zurückgekehrt wird). |
| `revert_secs` | 600 | — | Sekunden ohne Verkehr, bevor zum konfigurierten `endpoint`/`talkgroup` zurückgekehrt wird. 0 bleibt beim zuletzt ausgewählten Reflector. |
| `[[…reflector.hosts]]` | keiner | — | Eigene Reflector-Einträge, `talkgroup` (1–65535), `endpoint` (`host:port`) und optional `name`; sie haben Vorrang vor der heruntergeladenen Liste. Bis zu 256. |

Beispiel mit automatischer Auswahl:

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

## 11. Analog-FM

Der FM-Kanal ist der originale MMDVM-FM-Repeater, der auf dem SDR läuft und entweder über svxlink (RoLink) oder über
eine Legacy-USRP-Anbindung mit 8 kHz mit einem Netz verbunden ist. Der RF-Kanal beträgt 25 kHz.

### 11.1 `[channel.fm]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `callsign` | erforderlich | N0CALL | Stations-Callsign (1–23 druckbare Zeichen), verwendet in den Netz-Metadaten. Der Text der CW ID ist `keyers[0].text`. |

### 11.2 `[channel.fm.modem]`: MMDVM-FM-Repeater (jeder Schlüssel erforderlich)

| Schlüssel | Station | Bedeutung und Regeln |
|---|---|---|
| `mode` | simplex | `link`, `simplex` oder `duplex`. |
| `access` | carrier_and_ctcss | Wie der Eingang öffnet: `carrier`, `ctcss_delayed`, `carrier_and_ctcss` oder `ctcss_latch`. |
| `external_enabled` | true | Den Netz-Audiopfad aktivieren. |
| `rx_level` | 200 | Skalierung des RF-Eingangs, 256 / `rx_level`. Darf nicht 0 sein. Bei 200 ist Reserve vorhanden; bei 128 wurden starke Stationen übersteuert (Clipping). |
| `tx_level` | 128 | Ausgangsskalierung. Die Einheiten von svxlink-`tx_ceiling` setzen 128 voraus. |
| `rf_boost` / `external_boost` | 1 / 1 | Verstärkungsfaktoren beim Weiterleiten von RF- bzw. Netz-Audio. |
| `ctcss_frequency` | 103 | CTCSS-Ton als ganzzahlige Hz aus der MMDVM-Tabelle (103 = 103.5 Hz). Gültige Codes: 67 69 71 74 77 79 82 85 88 91 94 97 100 103 107 110 114 118 123 127 131 136 141 146 151 156 159 162 165 167 171 173 177 179 183 186 189 192 196 199 203 206 210 218 225 229 233 241 250 254. |
| `ctcss_high` / `ctcss_low` | 150 / 100 | Schwellwerte des CTCSS-Detektors, in Original-MMDVM-Einheiten. |
| `ctcss_level` | 16 | Pegel des gesendeten CTCSS-Tons: etwa 400 Hz Deviation auf dieser Station. |
| `squelch_high` / `squelch_low` | 111 / 105 | Schwellwerte des Carrier-Squelch auf der RSSI-Metrik. Der Squelch öffnet oder schließt nach 4 aufeinanderfolgenden Messwerten jenseits eines Schwellwerts. Mit `cos_invert = true` öffnet er bei oder unter `squelch_low` und schließt bei oder über `squelch_high`. |
| `cos_invert` | true | Kehrt den Vergleich um. Auf diesem Board misst die Metrik Rauschen, daher ergibt ein starkes Signal einen *niedrigen* Wert. Gemessener Idle Floor: 115.3. |
| `max_deviation` | 0 | Schwellwert für das Blanking bei Over-Deviation × 128; 0 schaltet es aus. Ersetzt übermäßiges Audio durch einen Bleep und Stille; es ist **kein** Limiter. Es beweist nichts über die RF-Deviation. |
| `timeout_level` | 20 | Pegel des Timeout-Tons und des Blanking-Bleeps. |
| `callsign_at_start` / `callsign_at_end` / `callsign_at_latch` | false | Wann die CW ID gesendet wird. |

**`[channel.fm.modem.timers]`**: alle Schlüssel erforderlich, in Millisekunden; 0 deaktiviert einen Timer.

| Schlüssel | Station | Bedeutung |
|---|---|---|
| `callsign` | 600000 | CW-ID-Intervall (10 min). |
| `timeout` | 180000 | Transmit Timeout (3 min). |
| `holdoff` | 0 | Hold-off-Timer. |
| `kerchunk` | 250 | Kerchunk-Timer. |
| `ack_min` / `ack_delay` | 1000 / 500 | Acknowledgement-Timer. |
| `hang` | 300 | Repeater Hang Time. |

Die genaue MMDVM-Semantik von `holdoff`, `kerchunk` und den Acknowledgement-Timern ist die des originalen
MMDVM-FM-Controllers. Diese Codebasis dokumentiert sie nicht weiter.

**`[[channel.fm.modem.keyers]]`**: genau drei Blöcke, in dieser Reihenfolge: Callsign, RF Acknowledgement,
Network Acknowledgement.

| Schlüssel | Station | Bedeutung |
|---|---|---|
| `text` | N0CALL / K / R | CW-Text, bis zu 255 Zeichen; unbekannte Zeichen werden übersprungen. |
| `speed_wpm` | 20 / 18 / 22 | Geschwindigkeit, darf nicht 0 sein. |
| `frequency_hz` | 1000 / 900 / 1100 | Tonfrequenz, 1–24000. |
| `high_level` / `low_level` | 0 / 0 | Tonpegel des Keyers in Original-MMDVM-Einheiten. Die Codebasis dokumentiert sie nicht genau. |

### 11.3 `[channel.fm.worker]`: Signalpfad (jeder Schlüssel erforderlich außer `squelch`)

| Schlüssel | Station | Bedeutung |
|---|---|---|
| `symbol_deviation` | 10.0 | Skalierung der FM-Modulation. Muss > 0 sein. |
| `rx_dc_block` | true | DC-Blocker vor der Demodulation. |
| `power_calibration` | 0 | Offset, der zum RSSI-dB-Messwert addiert wird. |
| `tx_peak` | 0.5 | Spitzenwert der Basisband-Hüllkurve, 0 < x ≤ 0.8. |
| `host_tx_gain` | 1.0 | Audioverstärkung RF → Netz. Muss > 0 sein. |
| `host_rx_gain` | 9.0 | Audioverstärkung Netz → RF. Muss > 0 sein. Auf dieser Station sendet svxlink flaches, unbegrenztes Audio, und nexus2 übernimmt die Verarbeitung. |
| `pre_emphasis` / `de_emphasis` | false / false | Host-seitige Filter. Hier aus, weil die svxlink-Optionen unten diese Arbeit übernehmen. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Wie bei DMR. |

**`[channel.fm.worker.squelch]`** (optional; jeder Schlüssel hat einen Standardwert):

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `mode` | fixed | fixed | `fixed`: die obigen Modem-Schwellwerte entscheiden. `shadow`: nur messen und melden. `adaptive`: ein Noise-Floor-Tracker entscheidet. `adaptive` erfordert `squelch_high = 1`, `squelch_low = 0` und `cos_invert = false`; das wird beim Start geprüft, nicht von `doctor`. |
| `open_db` / `close_db` | 10 / 5 | — | SNR, die zum Öffnen und zum Offenbleiben nötig ist (0–60; close < open). Verwendet von `adaptive`. |
| `quiet_open_db` / `quiet_close_db` | 6 / 3 | — | Quieting, das zum Öffnen und zum Offenbleiben nötig ist (0–60; close < open). |

### 11.4 `[channel.fm.svxlink]`: Bridge zu svxlink / RoLink

Diese Bridge sendet rohes 16-kHz-PCM über UDP, mit Squelch und PTT über PTYs. Sie schließt `[channel.fm.usrp]` aus:
genau eine von beiden konfigurieren.

| Schlüssel | Standard | Station | Bedeutung und Regeln |
|---|---|---|---|
| `rx_audio` | erforderlich | 127.0.0.1:40100 | Wohin RF-Audio gesendet wird. Entspricht svxlink `[Rx] AUDIO_DEV=udp:…`. |
| `tx_audio_bind` | erforderlich | 127.0.0.1:40101 | Lokaler Socket für Netz-Audio von svxlink `[Tx] AUDIO_DEV=udp:…`. Muss sich von `rx_audio` unterscheiden. |
| `ptt_pty` | erforderlich | …/var/run/ptt | svxlink `[Tx] PTT_PTY`. |
| `squelch_pty` | erforderlich | …/var/run/sql | svxlink `[Rx] PTY_PATH` (`SQL_DET=PTY`). Muss sich von `ptt_pty` unterscheiden. |
| `talker_file` | keiner | …/var/run/talker | Datei, in die svxlink den aktuellen Talker schreibt; wird im Dashboard angezeigt. |
| `tg_file` | keiner | …/var/run/tg | Datei mit der ausgewählten Talkgroup. |
| `conf` | keiner | …/svxlink.conf | svxlinks eigene Konfiguration, einmal beim Start gelesen, für den angezeigten Netznamen, das Callsign und die Talkgroups. |
| `network` | vom Reflector-Host | RoLink | Anzeigename. |
| `jitter_ms` | 75 | 75 | Prebuffer für eingehendes Audio (0–500). |

**Empfangsverarbeitung (RF → Netz).** Jede Stufe ist standardmäßig aus; die Station verwendet den empfohlenen Satz.

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `rx_de_emphasis_hz` | 0 (aus) | 300.0 | De-Emphasis-Eckfrequenz, 0 oder 50–2000 Hz. |
| `rx_gate_depth_db` | 0 (aus) | 30.0 | Tiefe des Downward Expanders, der dem Noise Floor folgt (0–60). Die folgenden Schlüssel gelten nur, wenn dieser Wert > 0 ist. |
| `rx_gate_open_db` / `rx_gate_close_db` | 10 / 5 | — | Pegel über dem Floor, der als Sprache gilt (3–40), bzw. der sie hält (1…open). |
| `rx_gate_hold_ms` / `rx_gate_attack_ms` / `rx_gate_release_ms` | 120 / 5 / 60 | — | Gate-Timing (Hold ≤ 2000, Attack 1–200, Release 5–2000). |
| `rx_floor_rise_db_per_s` | 5.0 | — | Wie schnell die Schätzung des Floor steigen darf (0.1–60). |
| `rx_level_target_dbfs` | aus | −21.0 | Zielwert des Levelers, RMS der Sprache (−40…−6, unterhalb der Peak-Obergrenze). Die nächsten Schlüssel gelten nur, wenn dieser gesetzt ist. |
| `rx_level_initial_gain_db` / `rx_level_min_gain_db` / `rx_level_max_gain_db` | 20 / −10 / 30 | — | Verstärkungen des Levelers (min −20…max, max 0–40). |
| `rx_level_rise_db_per_s` / `rx_level_fall_db_per_s` | 8 / 30 | — | Geschwindigkeit des Levelers (0.5–60 / 1–120). |
| `rx_peak_ceiling_dbfs` | aus | −6.0 | Obergrenze des Look-ahead-Limiters (−20…−0.5). |

**Sendeverarbeitung (Netz → RF):**

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `tx_pre_emphasis` | false | true | nexus2 wendet Pre-Emphasis an (+6 dB/Oktave oberhalb 300 Hz, Einheitsverstärkung bei 1 kHz). Dazu svxlink `[Tx1] PREEMPHASIS=0` setzen. |
| `tx_limiter` | false | true | Look-ahead-Peak-Limiter bei `tx_ceiling`. Dazu svxlink `LIMITER_THRESH=0` setzen. |
| `tx_ceiling` | 2048 | 2200 | Obergrenze des Limiters (256–3072). 1 Einheit ≈ 1.9 Hz Deviation bei `tx_level` 128, also 2200 ≈ 4.2 kHz. |

### 11.5 `[channel.fm.usrp]`: Legacy-USRP-Anbindung (Alternative zu svxlink)

Alle Schlüssel sind erforderlich. Die Anbindung überträgt Legacy-Audio mit 8 kHz.

| Schlüssel | Bedeutung |
|---|---|
| `bind` | Lokale UDP-`ip:port`. Sie darf keinen Port ungleich null mit einem früheren FM-Kanal teilen. |
| `peer` | Entfernte Unicast-`ip:port`, gleiche Adressfamilie wie `bind` und von ihr verschieden. |
| `loss_timeout_ms` | Nach dieser Zeit ohne Pakete wird die Netz-Sprache freigegeben. Muss > 0 sein. |

### 11.6 `[callsigns]`: Identitäten und MDC1200

Diese Tabelle wird nur von der svxlink-FM-Bridge verwendet. Sie wird beim Start geladen und mit `nexus2 callsigns`
neu erstellt.

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `enabled` | true | true | Die Identitäten überhaupt verwenden. |
| `database` | keiner | /opt/nexus-bs2/callsigns.csv | `callsign,mdc_id`-Datei. Sie muss beim Start lesbar sein, sonst schlägt der Start fehl. |
| `source` | keiner | RoLink node API | Operatorliste, die beim Neuaufbau verwendet wird: eine URL oder eine lokale Datei. |
| `selection` | active | active | `active` = innerhalb von `active_days` gehörte Operatoren; `all` = jeder jemals gelistete Operator. |
| `active_days` | 30 | 60 | Rückblickzeitraum (1–3650). |
| `mdc_encode` | false | true | Die MDC1200-ID des Netz-Talkers vor dem Netz-Audio auf Analog-FM senden. Nur ausgehend: auf RF empfangenem MDC wird niemals vertraut. |
| `unknown` | RoLink | ROLINK | Bezeichnung, die für einen nicht in der Datenbank enthaltenen Talker angesagt wird. Leer sendet nichts. |

## 12. Zurückgestellte Betriebsarten

Kanäle mit `dstar`, `ysf`, `nxdn` und `pocsag` werden geparst und erscheinen in `plan`/`doctor`, aber ein Live-Start
weist sie zurück, solange sie aktiviert sind. Ihre Tabellen akzeptieren `tx_delay` (alle vier), `low_deviation` und
`hang_seconds` (YSF, Standard 4) sowie `hang_seconds` (NXDN, Standard 5). Sie existieren nur für die Planung.

## 13. `[dashboard]`, `[network]`, `[touch]`

### `[dashboard]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `listen` | 127.0.0.1:8872 | 0.0.0.0:8080 | HTTP- und WebSocket-Adresse. Eine Nicht-Loopback-Adresse erfordert `auth.enabled = true`. |
| `id_database_url` | https://radioid.net/static/user.csv | — | Quelle der Callsign-/Namens-/Länderdatenbank (nur https). Wird aktualisiert, wenn sie älter als 7 Tage ist. |
| `id_database_dir` | `<config dir>/ids` | — | Wo diese Datenbank gespeichert wird. |
| `host_lists.dmr_url` | https://www.pistar.uk/downloads/DMR_Hosts.txt | — | DMR-Master-Liste (Pi-Star-Format) für die Settings-Seite. |
| `host_lists.p25_url` | https://refcheck.radio/api/hostfile-gate/fetch/p25/ | — | P25-Reflector-Liste (P25Hosts.txt-Format): das DVRef-Register, bereitgestellt von RefCheck.Radio. Wird auch von P25 `auto_select` verwendet. |
| `host_lists.p25_token` | keiner | — | Ihr persönliches RefCheck.Radio-Token: Geben Sie Ihr Callsign auf https://hostfiles.refcheck.radio ein, um eines zu erhalten. Ohne Token wird die P25-Liste nicht heruntergeladen (Ihre manuell eingetragenen `hosts` funktionieren weiterhin). Deren Nutzungsbedingungen erlauben einen Download pro Stunde; die Liste nennt DVRef als Quelle. Wird aus Share-Exporten entfernt. |
| `host_lists.refresh_hours` | 24 | — | Lädt die Listen nach so vielen Stunden erneut herunter (1–720). Die zwischengespeicherten Kopien liegen neben der ID-Datenbank. |
| `auth.enabled` | true | true | HTTP-Basic-Authentifizierung auf jeder Seite, jeder API-Route und jedem WebSocket. |
| `auth.username` | admin | gesetzt | 1–64 druckbare Zeichen, kein `:`. |
| `auth.password` | nexus | gesetzt | 1–256 Zeichen. **Ändern Sie den Standardwert.** |

### `[network]`

| Schlüssel | Standard | Station | Bedeutung |
|---|---|---|---|
| `connect_timeout_secs` | 10 | 10 | Wird validiert (1–300), aber derzeit von keiner Anbindung verwendet. |
| `brew` | keiner | gesetzt | TETRA-Core-Anbindung, §8.5. |

### `[touch]`: Panel Nexus-BS Touch

Diese Tabelle wird nur vom Panel gelesen, das sie innerhalb von 5 s nach einer Änderung neu einliest.

| Schlüssel | Standard | Bedeutung |
|---|---|---|
| `backlight` | 100 | % verwendete Hintergrundbeleuchtung (1–100). |
| `screen_timeout_secs` | 300 | Sekunden ohne Bedienung bis zum Bildschirmschoner; 0 = immer an (maximal 86400). |
| `screensaver` | splash | `splash` = das Boot-Logo, statisch und abgedunkelt; `blank` = schwarz mit ausgeschalteter Hintergrundbeleuchtung. |
| `screensaver_dim` | 30 | % Helligkeit des abgedunkelten Logos. |
| `screensaver_backlight` | 25 | % Hintergrundbeleuchtung, während das Logo angezeigt wird. |
| `wake_on_touch` | true | Eine Berührung weckt den Bildschirm. Diese Berührung wird verbraucht und betätigt niemals ein Bedienelement. |
| `wake_on_traffic` | true | Ein neuer Call oder ein neuer Last-Heard-Eintrag weckt den Bildschirm. |

Wenn ein Timeout gesetzt ist, muss mindestens eine der beiden Weck-Optionen eingeschaltet sein. Die Steuerung der
Hintergrundbeleuchtung erfordert `/sys/class/backlight` und Schreibrechte; ohne diese dunkelt das Panel nur das Bild ab.

## 14. Kommandozeile

`--config <file>` ist standardmäßig `nexus2.toml`.

| Befehl | Zweck |
|---|---|
| `nexus2 plan` | Den Frequenzplan ohne Hardware anzeigen. |
| `nexus2 doctor [--live]` | Alles validieren, was offline geprüft werden kann. `--live` fügt die Anforderungen des Live-Starts hinzu (§6). |
| `nexus2 plan-risks --occupied-half-bandwidth-hz N --rx-bandwidth-hz N --tx-bandwidth-hz N [--sample-rate N]` | Offline-Geometrie von DC, Image und Intermodulation des Plans. |
| `nexus2 init [--defaults]` | Eine neue Datei erstellen (TX gesperrt). Weigert sich, eine vorhandene zu überschreiben. |
| `nexus2 callsigns --out FILE [...]` | Die Callsign-/MDC1200-Datenbank aus `[callsigns]` neu erstellen. |
| `nexus2 run --live [--allow-rf-tx]` | Die Station selbst (der systemd-Dienst). |
| `nexus2 manage ...` | Die HTTPS-Management-API (separater Dienst). |

## 15. Rezepte

**Einen Carrier verschieben.** Ändern Sie sein `offset_hz` um ein Vielfaches von 25 kHz. Halten Sie 50 kHz Abstand zu
den Nachbarn und bleiben Sie innerhalb von ±(Sample Rate / 2 − 20 kHz). Führen Sie `nexus2 plan` und dann
`doctor --live` aus (TETRA muss eine Standardkodierung behalten), wenden Sie die Änderung an und starten Sie neu.

**Eine Betriebsart dauerhaft abschalten.** Setzen Sie `enabled = false` in ihrem `[[channel]]`. Sie behält ihren
Frequenzplatz. Bei DMR, P25 oder FM kann eine vorübergehende Abschaltung live über das Dashboard erfolgen; sie gilt bis
zum nächsten Neustart.

**Den TETRA-Packet-Data-Carrier aktivieren.** Setzen Sie `enabled = true` bei `tetra-data`, wobei seine Identität
gleich der des primären Kanals bleibt, und starten Sie neu. Der primäre Kanal behält seine
`[channel.tetra.packet_data]`.

**Den TX-Pegel einer Betriebsart anpassen.** Ändern Sie `tx_peak` dieser Betriebsart: `[channel.tetra.worker]`,
`[channel.dmr.worker]`, `[channel.p25.worker]` oder `[channel.fm.worker]`. Halten Sie ihn bei 0.8 oder darunter. Die
Summe aller Carrier teilt sich einen DAC, daher erhöht das Anheben einer Betriebsart die zusammengesetzte Spitze.
`sdr.tx_gain_db` und die Element-Verstärkungen verschieben **alle** Betriebsarten gemeinsam.

**Empfangspegel messen.** Setzen Sie `sdr.io_diagnostics = true`, starten Sie neu, sammeln Sie die
`RX_LEVEL`-Journalzeilen und setzen Sie den Wert dann wieder auf `false`.

**Das Dashboard-Passwort ändern.** Setzen Sie `dashboard.auth.password` und starten Sie den Funkdienst neu. Das
Touch-Panel meldet sich mit den Zugangsdaten aus derselben Datei am Dashboard an, liest sie aber nur einmal beim
eigenen Start; starten Sie daher auch `nexus-panel.service` neu.

## 16. Bekannte Einschränkungen und offene Punkte

Diese Punkte ergeben sich aus dem aktuellen Stand des Codes; sie werden genannt, damit sich niemand darauf verlässt:

- `network.connect_timeout_secs` wird validiert, aber keine Anbindung verwendet es.
- Die im Code dokumentierte TETRA-`tx_peak`-Grenze von 0.8 wird nicht erzwungen; erzwungen wird nur 0–1.0, wenn TX
  aktiviert ist.
- `packet_data.max_slots` außerhalb von 1–4 wird stillschweigend begrenzt.
- `sdr.driver` wählt das Gerät nicht aus (das tut `uri`). Ein falscher Treibername ändert nur das SX1255-spezifische
  Verhalten.
- `nexus2 callsigns` fällt stillschweigend auf Standardwerte zurück, wenn die Datei nicht geladen werden kann, und
  kann das SDR abfragen, wenn `[sdr]` auto verwendet.
- In dieser Codebasis nicht dokumentiert: die genauen Einheiten von DMR `receiver_delay`; was genau `stall_ms`
  auslöst; die Skalen von FM-Keyer `high_level`/`low_level` sowie der CTCSS-Schwellwerte/-Pegel (Original-MMDVM-Byte-Einheiten);
  der längste CW-Text, der hineinpasst.
- Das Neuladen von Einstellungen pro Kanal ohne Neustart existiert im Code, hat aber keinen produktiven Aufrufer. Nur
  der Ein/Aus-Umschalter für DMR, P25 und FM ist live.
