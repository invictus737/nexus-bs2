# Manualul de configurare Nexus-BS 2.0 (`nexus2.toml`)

Versiunea în engleză: [Manual_conf.md](Manual_conf.md)

Acest manual explică fiecare cheie a fișierului de configurare al stației, cum este verificat fișierul și cum ajung
modificările la stația aflată în funcțiune. A fost scris pe baza codului sursă (commit `d7fdec7` și ulterioare), nu
din memorie. Acolo unde codul nu lămurește un detaliu, manualul spune acest lucru în loc să ghicească.

Configurația stației aflate în funcțiune este referința pentru valorile recomandate. Singurul exemplu livrat cu Nexus-BS 2.0,
`examples/nexus2.example.toml`, folosește aceleași setări, cu identități generice (`N0CALL`, `CHANGE_ME`), planul de frecvențe al exemplului TETRA BlueStation și TX blocat. În tabelele de mai jos:

- **Implicit** este ceea ce folosește codul când cheia lipsește. „obligatoriu" înseamnă că nu există valoare implicită.
- **Stație** este valoarea din fișierul stației aflate în funcțiune. O liniuță înseamnă că cheia lipsește acolo.

Cuprins:
1. [Cum este citit fișierul](#1-cum-este-citit-fișierul)
2. [Aplicarea unei modificări](#2-aplicarea-unei-modificări)
3. [Frecvențele și planul de canale](#3-frecvențele-și-planul-de-canale)
4. [Permisiunea de TX și comutatorul RF](#4-permisiunea-de-tx-și-comutatorul-rf)
5. [`[sdr]`](#5-sdr-radioul-partajat)
6. [`[[channel]]`](#6-channel-chei-comune)
7. [`[channel.access]`](#7-channelaccess-politica-de-admitere)
8. [TETRA](#8-tetra)
9. [DMR](#9-dmr)
10. [P25](#10-p25)
11. [FM analogic](#11-fm-analogic)
12. [Moduri amânate](#12-moduri-amânate)
13. [`[dashboard]`, `[network]`, `[touch]`](#13-dashboard-network-touch)
14. [Linia de comandă](#14-linia-de-comandă)
15. [Rețete](#15-rețete)
16. [Limitări cunoscute și puncte deschise](#16-limitări-cunoscute-și-puncte-deschise)

---

## 1. Cum este citit fișierul

- **Format.** Fișierul este TOML 1.0. În cadrul unui tabel, ordinea cheilor este liberă. Un antet de tabel precum
  `[network.brew]` poate apărea oriunde în fișier, chiar și după intrările `[[channel]]`. Fișierul stației folosește
  acest lucru pentru a grupa fiecare legătură de rețea cu modul ei.
- **Strict.** Fiecare tabel respinge cheile necunoscute. O greșeală de scriere este o eroare fatală, de exemplu
  `config: unknown field `brightnes` near line 70`. Mesajele de eroare dau calea cheii și numărul liniei, niciodată
  valoarea, astfel încât secretele nu ajung în loguri.
- **Tabele obligatorii.** `[sdr]`, cel puțin un `[[channel]]`, `[dashboard]` și `[network]` sunt obligatorii.
  `[dashboard]` și `[network]` pot fi goale. `[touch]` și `[callsigns]` sunt opționale.
- **Forme ale valorilor.**
  - Un întreg este acceptat acolo unde se așteaptă un număr zecimal (`rx_gain_db = 30`).
  - Literalii hexazecimali, octali și binari funcționează pentru orice întreg (`nac = 0x293`), la fel și liniuțele de
    subliniere (`offset_hz = -50_000`).
  - Numerele negative trebuie scrise în zecimal.
- **Unități.** Frecvențele sunt în Hz. Timpii sunt în ms, cu excepția cazului în care numele cheii se termină în `_secs`.
- **Secrete.** Secretele sunt șiruri simple: `dashboard.auth.password`, `network.brew.password`,
  `channel.dmr.network.password` și `sdr.uri`. Ele sunt ascunse în ieșirea de depanare și în exportul cenzurat.
  Păstrați fișierul cu modul 0600. Dashboard-ul și API-ul de management îl scriu amândouă în acest fel.

## 2. Aplicarea unei modificări

**Procesul radio citește fișierul o singură dată, la pornire.** Editarea fișierului nu schimbă nimic până la
repornirea serviciului. Există două excepții:

| Ce | Intră în vigoare |
|---|---|
| `[touch]` | Panoul îl recitește în aproximativ 5 s. O valoare în afara domeniului revine la valoarea sa implicită. |
| Activarea sau dezactivarea unui canal **DMR, P25 sau FM** din comutatorul de pe dashboard sau de pe panou | Imediat, fără repornire. Starea este păstrată **doar în memorie** și nu este scrisă în fișier: după o repornire a serviciului se aplică din nou valoarea `enabled` din fișier. TETRA nu are comutator live. |

Orice altceva necesită o repornire. `[callsigns]` este de asemenea încărcat doar la pornire.

**Verificați înainte de aplicare:**

```sh
nexus2 --config nexus2.toml plan         # frequency plan, no hardware
nexus2 --config nexus2.toml doctor       # full static validation
nexus2 --config nexus2.toml doctor --live   # plus the checks a live start makes
```

`doctor` nu deschide niciodată SDR-ul sau rețeaua. Nu rezolvă DNS, nu leagă socket-uri și nu citește baza de date
`[callsigns]`, așa că un `doctor` fără erori nu garantează că legăturile de rețea se ridică.

**Pe stație** (API-ul de management de pe o stație de lucru; vedeți MANAGEMENT-API.md):

```sh
A="python3 scripts/nexus2-admin.py --url https://<station-ip>:9443 --connect-ip <station-ip>"
$A config > current.json                 # contains secrets: keep private
$A validate candidate.toml               # validate with the station's own checks
$A set-config candidate.toml --if-match <config_sha256 from status>
```

- `set-config` înlocuiește fișierul, păstrează o copie de siguranță, repornește radioul și revine la versiunea
  anterioară dacă procesul nu rămâne sănătos.
- Precondiția de hash refuză să suprascrie un fișier care s-a schimbat între timp.
- Repornirea întrerupe pentru scurt timp **fiecare mod**.
- Managerul validează cu propriul său binar. După o actualizare a radioului care adaugă chei noi, actualizați și
  managerul, altfel acesta le respinge (HTTP 422).
- Cât timp API-ul de management este instalat (fișierul marcaj `/opt/nexus-bs2/.management-enabled`), editorul de
  configurație propriu al dashboard-ului și formularul de setări refuză să salveze și răspund `use_management_api`.

## 3. Frecvențele și planul de canale

Un singur SDR poartă toate carrier-ele. Fiecare canal este un offset față de cele două local oscillators (LO):

```
RX frequency (uplink, terminals -> station)   = sdr.rx_lo_hz + offset_hz
TX frequency (downlink, station -> terminals) = sdr.tx_lo_hz + offset_hz
```

Există un singur `offset_hz` per canal, deci fiecare carrier are același duplex split: `tx_lo_hz - rx_lo_hz`.

**Regulile planului.** Acestea sunt verificate pentru **toate** canalele, inclusiv cele dezactivate:

| Regulă | Eroare dacă este încălcată |
|---|---|
| Între 1 și 7 canale în total | numărul de canale |
| `offset_hz` este un multiplu exact de 25 000 | `channel[i].offset_hz: must be an exact multiple of 25000` |
| Oricare două canale la cel puțin 50 000 Hz distanță (o celulă goală de 25 kHz între carrier-e) | eroare de plan |
| `abs(offset_hz) + 20 000 < sample_rate / 2` | "insufficient Nyquist/filter margin" |
| Plăcile SX1255 (`sx`, `mucell`): niciun canal **activat** la `offset_hz = 0`, deoarece ar sta pe DC | "carrier would sit on DC; shift rx_lo_hz/tx_lo_hz by a 25 kHz multiple instead" |

- Passband-ul analogic necesar este `2 × (max |offset_hz| + 20 000)`. Pentru stație acesta este
  2 × (175 000 + 20 000) = 390 kHz.
- **Sample rate.** Când `sdr.sample_rate` lipsește sau este 0, stația întreabă radioul ce sample rate-uri suportă. Le
  păstrează pe cele care sunt de cel mult 1 MHz, multiplu de 50 kHz și mai largi decât passband-ul necesar. Apoi o
  alege pe cea mai mică dintre cele de cel puțin dublul passband-ului, altfel pe cea mai largă utilizabilă. Stația
  ajunge la 600 kS/s.
- **Codificarea frecvenței TETRA.** Un carrier TETRA trebuie să fie și un canal TETRA standard:
  - Frecvența TX trebuie să fie pe rasterul de 25 kHz, opțional decalată cu +6.25, −6.25 sau +12.5 kHz.
  - Duplex spacing-ul RX–TX trebuie să fie unul dintre duplex spacing-urile standard pentru bandă. În banda de 400 MHz
    acestea sunt 10, 7, 8, 5 sau 9.5 MHz. Stația folosește 7 MHz.
  - Altfel eroarea este `tetra: frequencies have no supported standard repeater encoding`.
  - `plan` nu verifică acest lucru; `doctor` și o pornire live îl verifică.

**Planul stației** (LO-uri 431.2875 / 438.2875 MHz):

| Canal | offset_hz | RX MHz | TX MHz | Stare |
|---|---|---|---|---|
| tetra-data | −50 000 | 431.2375 | 438.2375 | dezactivat |
| dmr | +25 000 | 431.3125 | 438.3125 | activat |
| tetra | +75 000 | 431.3625 | 438.3625 | activat |
| p25 | +125 000 | 431.4125 | 438.4125 | activat |
| analog (FM) | +175 000 | 431.4625 | 438.4625 | activat |

Pentru a muta întreaga stație, modificați ambele LO-uri cu aceeași valoare. Pentru a muta un singur carrier,
modificați-i `offset_hz`, respectați regulile grilei și verificați cu `nexus2 plan`.

## 4. Permisiunea de TX și comutatorul RF

Trei lucruri separate decid dacă stația face TX:

1. **`sdr.tx_inhibited`** (configurație, implicit în cod `true`). Când este true, nu se pregătește deloc nicio cale de
   TX. Fișierul stației setează `false`. `nexus2 init` scrie întotdeauna `true`.
2. **`--allow-rf-tx`** (linia de comandă a `run --live`). Aceasta este doar permisiunea procesului. Un fișier cu
   `tx_inhibited = false` este refuzat la pornire fără ea. Unitatea de serviciu o transmite.
3. **Comutatorul RF global** (RF KILL / RESTORE RF pe dashboard și pe panou). Acesta **nu este o cheie de
   configurație**.
   - Starea sa este păstrată în `~/.local/state/nexus2/rf-<hash of the config path>`, deci supraviețuiește
     repornirilor. OFF oprește RX și TX pentru toate modurile.
   - Doar un RF KILL dat de operator, sau o eroare fatală la oprirea curată a radioului, memorează OFF.
   - O pornire eșuată păstrează ON și reîncearcă singură, după 2 s și apoi cu o pauză crescătoare de până la 60 s. Un
     exemplu este un nume de rețea care încă nu se rezolvă în timpul boot-ului.
   - Acesta este un control software, nu o interblocare hardware.

## 5. `[sdr]`: radioul partajat

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `driver` | `"auto"` | — | Numele driverului SoapySDR, sau `auto`/gol pentru a folosi cel mai bun radio găsit. **Nu** este folosit pentru a deschide dispozitivul: pentru asta se folosește `uri`. El doar îi spune stației dacă placa este un SX1255 (`sx`, `mucell`). Aceasta afectează regula DC, valoarea implicită `tx_lead_quanta` și un avertisment despre HAT-ul montat. Ordinea de preferință în modul auto: plăcile SX1255 (`sx` înaintea `mucell`), Lime, USRP B2xx, alte USRP, Pluto, orice altceva. |
| `uri` | `"auto"` | — | Argumentele dispozitivului SoapySDR (1–256 caractere), sau `auto` pentru a le prelua de la radioul găsit. |
| `sample_rate` | `0` (auto) | — | Sample rate-ul complex al întregului dispozitiv. Trebuie să fie un multiplu pozitiv de 50 000 și cel mult 1 000 000. Vedeți §3. |
| `rx_lo_hz` | obligatoriu | 431287500 | RX local oscillator (LO). |
| `tx_lo_hz` | obligatoriu | 438287500 | TX local oscillator (LO). |
| `rx_gain_db` | obligatoriu | 30.0 | Câștigul global RX (−100…100), aplicat înaintea câștigurilor pe elemente. |
| `tx_gain_db` | obligatoriu | 0.0 | Câștigul global TX (−100…100). |
| `rx_gains_db.<element>` | niciunul | LNA 42, PGA 16 | Câștiguri RX pe elemente, aplicate după câștigul global. Numele elementelor provin de la driver. |
| `tx_gains_db.<element>` | niciunul | DAC 9, MIXER 30 | Câștiguri TX pe elemente. |
| `rx_bandwidth_hz` / `tx_bandwidth_hz` | auto | — | Lățimea filtrului analogic. Dacă este setată, trebuie să fie **mai mare decât** passband-ul necesar (§3). Când lipsește, se folosește domeniul driverului. |
| `rx_channel` / `tx_channel` | 0 | — | Indexul canalului hardware la radiourile cu mai multe canale. |
| `rx_antenna` / `tx_antenna` | în funcție de placă | — | Portul de antenă. Valorile implicite sunt SX1255 `RX`/`TX`, Lime USB `LNAL`/`BAND1`, alte Lime `LNAW`/`BAND2`, Pluto `A_BALANCED`/`A`, USRP `TX/RX`. |
| `rx_stream_args` / `tx_stream_args` | gol | — | Argumente suplimentare de stream SoapySDR (`key = "value"`). |
| `settings.<key>` | gol | — | Setări SoapySDR la nivelul întregului dispozitiv. Plăcile SX1255 primesc `PA = "AUTO"` dacă nu este setat aici. |
| `ppm` | 0.0 | — | Corecție de frecvență pentru ambele căi (−100…100). O valoare nenulă necesită un driver care suportă corecția, altfel deschiderea radioului eșuează. |
| `tx_lead_quanta` | 8 pe SX1255, 3 în rest | — | Câte blocuri de 1 ms de eșantioane TX sunt pregătite în avans față de ceasul radioului (1–8). 8 este măsurat pe SX1255; 3 nu este măsurat pe radiourile USB. Creșteți-l primul dacă logul arată "TX late block skipped". |
| `sx1255_rx_pll_bw_khz` | driver (300) | — | Lățimea de bandă a buclei PLL RX a SX1255: 75, 150, 225 sau 300. Respinsă pe alte radiouri. |
| `sx1255_tx_pll_bw_khz` | driver (150) | 300 | Lățimea de bandă a buclei PLL TX a SX1255. 300 dă cel mai mic close-in phase noise pe carrier. |
| `tx_inhibited` | **true** | false | Vedeți §4. |
| `io_diagnostics` | false | false | Ajutor de măsurare: ferestre de nivel RX și contoare de temporizare I/O, plus blocul `rf_io` din dashboard. Scrie aproximativ 11 linii de jurnal pe secundă, așa că lăsați-l oprit în funcționare normală. Nu modifică RF-ul. |
| `pipeline_diagnostics` | false | — | Instrumentare experimentală a cozilor și canalelor. |

## 6. `[[channel]]`: chei comune

Fiecare bloc `[[channel]]` este un carrier. Tabelul modului său trebuie să îl urmeze, împreună cu `[channel.access]`
pentru DMR, P25 și FM.

| Cheie | Implicit | Semnificație și reguli |
|---|---|---|
| `id` | obligatoriu | Nume unic, 1–64 caractere din `A-Z a-z 0-9 - _`. Dashboard-ul, API-ul de control și un `cell` de packet data se referă la canale prin el. |
| `mode` | obligatoriu | `tetra`, `dmr`, `p25` sau `fm`. `dstar`, `ysf`, `nxdn` și `pocsag` sunt de asemenea interpretate, dar o pornire live le refuză dacă sunt activate (§12). Tabelul corespunzător (`[channel.tetra]`, …) este obligatoriu, iar orice alt tabel de mod este o eroare. |
| `enabled` | true | Pornit/oprit administrativ. Un canal dezactivat contează în continuare pentru regulile planului și pentru limita de 7 carrier-e. |
| `offset_hz` | obligatoriu | Offset față de ambele LO-uri (§3). |

**Ordinea blocurilor `[[channel]]`.** Ordinea nu schimbă nicio frecvență. Ea stabilește indexul sub care este afișat
un canal în mesajele de eroare (`channel[2].…`) și în formularul de setări al dashboard-ului.

- Nivelul TX TETRA este preluat de la primul canal TETRA activat care are un `[channel.tetra.worker]`
  (`tx_peak`). Păstrați canalul TETRA primar primul.
- Conflictele de port USRP sunt verificate doar față de canalele anterioare.
- Reordonarea, adăugarea sau eliminarea canalelor necesită o repornire.

**Cerințe pentru pornirea live.** Acestea depășesc ceea ce verifică `doctor` fără `--live`:

- Exact un canal TETRA primar activat, astfel încât stația să nu funcționeze fără TETRA.
- Orice alt canal TETRA activat trebuie să fie carrier-ul de packet data al acelei celule.
- Un canal DMR activat necesită `worker`, `network` și `network.runtime`.
- Un canal P25 activat necesită `worker` și `reflector`.
- Un canal FM activat necesită `modem`, `worker` și fie `svxlink`, fie `usrp`.
- Fiecare canal DMR, P25 sau FM activat necesită `[channel.access]`.

## 7. `[channel.access]`: politica de admitere

Acest tabel decide ce trafic lasă să treacă un mod. Implicit nu se admite nimic și nu acordă niciodată TX fizic.
`mode` trebuie să fie egal cu modul canalului. TETRA nu are tabel de acces.

| Mod | Cheie | Semnificație |
|---|---|---|
| dmr | `rf_slots = [TS1, TS2]` (obligatoriu) | Admite trafic RF (voce, date, CSBK) pe fiecare timeslot. |
| dmr | `network_slots = [TS1, TS2]` (obligatoriu) | Admite trafic din rețea către RF pe fiecare timeslot. |
| dmr | `protected_voice` (implicit false) | Admite voce care poartă flag-ul de privacy/encryption (PI). Când este false, este refuzată în ambele direcții. |
| p25 | `rf` (obligatoriu) | Admite apeluri RF. |
| p25 | `network` (obligatoriu) | Admite trafic din reflector către RF. |
| fm | `selected` (obligatoriu) | Intrarea MMDVM "FM mode selected" a logicii de repeater. Nu este accesul prin carrier/CTCSS. |

## 8. TETRA

### 8.1 `[channel.tetra]`

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `mcc` | obligatoriu | 901 | Mobile country code (MCC) (0–1023). |
| `mnc` | obligatoriu | 9999 | Mobile network code (MNC) (0–16383). |
| `location_area` | obligatoriu | 2 | Location area de bază (0–16383). |
| `colour_code` | obligatoriu | 1 | Colour code (0–63). |
| `rotate_location_area_on_start` | true | — | Când este true, fiecare pornire difuzează un LA diferit, derivat din `location_area`, astfel încât terminalele atașate celulei (camped) se reînregistrează după o repornire. Setați false pentru a difuza exact `location_area`. |
| `implicit_group_affiliation` | true | — | Un apel sau un floor request de la un terminal care nu este cunoscut ca atașat la acel group îl atașează, în loc să fie refuzat(ă). |
| `station_name` | `"Nexus-BS 2.0"` | — | Textul mesajului periodic Home Mode Display afișat pe terminale. Gol îl dezactivează. |
| `timezone` | fusul orar al gazdei | — | Numele zonei IANA (de ex. `Europe/Bucharest`) pentru ceasul difuzat de celulă. Gol dezactivează ceasul. Un nume invalid face ca `doctor` să eșueze. |
| `role` | `"primary"` | — / `packet_data` | `packet_data` face din acesta un carrier secundar de date (§8.4). |
| `cell` | niciunul | — / `tetra` | Doar pentru `role = "packet_data"`: `id`-ul canalului primar. |

### 8.2 `[channel.tetra.packet_data]`: IP peste TETRA (doar primar)

Când este prezent, celula oferă SNDCP packet data. Terminalele primesc adrese din pool și ajung la stație printr-o
interfață tun Linux.

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `ipv4_pool` | obligatoriu | `10.44.0.0/24` | CIDR cu prefix /16…/30 și fără biți de host. Primul host este gateway-ul pe interfața tun (10.44.0.1); restul sunt alocate terminalelor. |
| `tun` | `"tetra0"` | — | Numele interfeței tun (1–15 caractere). |
| `mtu` | 576 | — | Dimensiunea N-PDU și MTU-ul interfeței (128–1500). |
| `header_compression` | true | false | Acordă header compression RFC 1144 (Van Jacobson) când un terminal o cere. Motorola MXP600 renunță la activare când aceasta este acordată, așa că stația o ține oprită. |
| `max_slots` | 4 | — | Numărul maxim de timeslot-uri pe care un terminal le poate folosi pe carrier-ul de date. Valorile în afara 1–4 sunt **readuse fără avertisment în intervalul 1–4**, nu respinse. |
| `advertise` | true | — | Anunță "SNDCP available" în informațiile de sistem. Când este oprit, serviciul răspunde totuși terminalelor care încearcă. |
| `advertise_advanced_link` | true | — | Anunță "advanced link supported". |

### 8.3 `[channel.tetra.worker]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `tx_peak` | 0.5 | 0.8 | Envelope-ul de vârf în baseband al carrier-ului TETRA, pe aceeași scară ca `tx_peak` pentru DMR, P25 și FM. 0.5 dă același vârf ca DMR; aproximativ 0.79 dă aceeași putere medie, cu vârfuri cu 4 dB mai mari. Păstrați-l la 0.8 sau mai puțin. Codul respinge doar valorile din afara 0–1.0, și doar când TX este activat. |

### 8.4 Carrier-ul de packet data (`role = "packet_data"`)

Un al doilea carrier TETRA al aceleiași celule, cu packet data pe toate cele patru timeslot-uri și fără control
channel. Terminalele sunt trimise pe el prin channel allocation de pe carrier-ul primar. Reguli:

- `cell` trebuie să numească un canal TETRA primar existent, iar acel primar trebuie să aibă
  `[channel.tetra.packet_data]`.
- `mcc`, `mnc`, `location_area` și `colour_code` trebuie să fie egale cu cele ale primarului.
- Un carrier de packet data nu are propriul tabel `packet_data` și există cel mult unul per primar.
- Frecvențele sale trebuie să fie în banda primarului, cu același duplex spacing.
- Când este dezactivat (`enabled = false`), celula păstrează packet data doar pe carrier-ul primar.

### 8.5 `[network.brew]`: legătura cu core-ul TETRA (Brew / TetraPack)

Acest tabel necesită cel puțin un canal TETRA activat.

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `host` | obligatoriu | core.tetrapack.online | Numele de host sau IP-ul serverului. |
| `port` | 443 | 443 | Portul serverului. |
| `tls` | true | true | Folosește TLS (wss/https). |
| `username` / `password` | niciunul | setat | Autentificare HTTP Digest. Fie ambele, fie niciunul. |
| `reconnect_delay_secs` | 15 | 15 | Pauza înainte de reconectare (1–300). |
| `jitter_initial_latency_frames` | 0 | — | Delay inițial suplimentar de playout, în frames, pentru vocea din rețea. |
| `feature_sds_enabled` | true | — | Transmite mesajele SDS între terminalele locale și cele din rețea. |
| `feature_rssi_export` | false | — | Raportează RSSI către server. |
| `whitelisted_ssis` | niciunul | — | Permite apeluri din rețea doar cu aceste SSI-uri de la distanță (până la 4096 de intrări, fiecare cel mult 0xFFFFFF). |

Cheile vechi `network.brew_endpoint` și `network.brew_token` sunt încă interpretate, dar `doctor` și o pornire live le
resping. Folosiți `[network.brew]` în locul lor.

## 9. DMR

### 9.1 `[channel.dmr]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `id` | obligatoriu | 1234567 | Repeater ID-ul pe 24 de biți pe RF (1–16777215). Este și ID-ul de rețea, cu excepția cazului în care este setat `network.id`. |
| `colour_code` | obligatoriu | 1 | Colour code (0–15). |

### 9.2 `[channel.dmr.worker]`: modem nativ (toate cheile obligatorii)

| Cheie | Stație | Semnificație și reguli |
|---|---|---|
| `symbol_deviation` | 10.0 | Scalarea FM a simbolurilor 4FSK, folosită pentru condiționarea RX și pentru TX. Trebuie să fie > 0. |
| `tx_level_q15` | −13056 | Câștigul de deviation TX în Q15 cu semn. −13056 = −102 × 128, nivelul SDR de referință. O valoare negativă inversează deviation-ul. |
| `tx_peak` | 0.5 | Envelope-ul de vârf în baseband, 0 < x ≤ 0.8. |
| `power_calibration` | 0 | Offset adăugat la citirea RSSI în dB. |
| `receiver_delay` | 3 | Delay-ul per slot al receptorului repeater-ului. Unitatea sa exactă nu este documentată în acest cod; păstrați 3. |
| `marker_offset_numerator` / `marker_offset_denominator` | 720 / 1 | Corecția dintre markerul de temporizare TX și RX-ul condiționat, în eșantioane la 24 kS/s: 720 = 30 ms. Numitorul nu trebuie să fie 0. |
| `maximum_input_age_ms` | 100 | Cea mai veche intrare RX pe care worker-ul o mai acceptă. Trebuie să fie > 0. |
| `stall_ms` | 5000 | Timeout-ul de blocare (stall) a worker-ului. Trebuie să fie > 0. |
| `hang_frames` | 51 | Hang frames trimise după un terminator de apel (0–60). |

### 9.3 `[channel.dmr.network]`: legătura MMDVM homebrew

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `profile` | obligatoriu | brandmeister | `brandmeister`, `dmrplus`, `tgif`, `freedmr` sau `custom` (un master propriu). Toate vorbesc protocolul MMDVM homebrew; doar profilul BrandMeister adaugă verificările asupra `version`/`software` de mai jos. Pagina Settings oferă master-ele rețelei alese din lista descărcată (vezi `[dashboard.host_lists]`). |
| `endpoint` | obligatoriu | 2262.master.brandmeister.network:62031 | `host:port` al master-ului, rezolvat când pornește canalul. |
| `password` | obligatoriu | setat | Parola de autentificare (1–256 octeți). |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62002 | Adresa UDP locală. Portul 0 lasă sistemul să aleagă. |
| `callsign` | obligatoriu | N0CALL | 1–8 caractere. |
| `id` | `[channel.dmr] id` | 123456701 | ID-ul de rețea. ID-ul efectiv trebuie să fie > 1000. |
| `name` | din host | — | Numele afișat pe dashboard. |
| `retry_ms` | 5000 | 1000 | Intervalul de reîncercare a autentificării. Trebuie să fie > 0. |
| `timeout_ms` | 15000 | 60000 | Timeout-ul sesiunii. Trebuie să fie mai mare decât `retry_ms`. |

### 9.4 `[channel.dmr.network.runtime]`: datele stației și politica sesiunii

Toate cheile sunt obligatorii, cu excepția `rf_inactivity_timeout_ms`.

| Cheie | Stație | Semnificație și reguli |
|---|---|---|
| `power`, `latitude`, `longitude`, `height` | 0, 0.0, 0.0, 0 | Datele stației trimise la autentificare: putere 0–99, latitudine ±90, longitudine ±180, înălțime 0–999. |
| `location` | "" | Până la 20 de caractere (textul mai lung este tăiat). |
| `description` | "Nexus-BS 2.0 hotspot" | Până la 19 caractere (textul mai lung este tăiat). |
| `url` | "" | Până la 124 de caractere. |
| `version` | 20260916_Nexus | Cu BrandMeister trebuie să fie `YYYYMMDD_…`. |
| `software` | MMDVM_Nexus | Cu BrandMeister trebuie să înceapă cu `MMDVM`, altfel master-ul refuză autentificarea. |
| `options` | "" | Șirul de opțiuni trimis master-ului, de ex. `TS1=1;TS2=1`. |
| `retention_ms` | 250 | Retenția vocii din rețea. Trebuie să fie > 0. |
| `rf_inactivity_timeout_ms` | 5000 | Timeout-ul de inactivitate RF. Omiteți-l sau setați o valoare > 0. |
| `embedded_lc_only` | true | true = trimite doar embedded link control pe vocea din rețea; false = păstrează datele embedded primite. |

## 10. P25

### 10.1 `[channel.p25]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `nac` | obligatoriu | 0x293 | Network Access Code (NAC), 0x000–0xFFF. Scrieți-l în hex, așa cum este programat în stațiile radio. Dashboard-ul îl afișează în zecimal (659). |

### 10.2 `[channel.p25.worker]`: modem nativ (toate cheile obligatorii)

| Cheie | Stație | Semnificație și reguli |
|---|---|---|
| `symbol_deviation`, `tx_level_q15`, `tx_peak` | 10.0, −13056, 0.5 | Ca la DMR (§9.2). |
| `duplex` | true | TX duplex. `tx_hang` funcționează doar în duplex. |
| `tx_delay` | 0 | Unități MMDVM originale: 500 ms + valoare × 10 ms, limitat la 1 s. |
| `tx_hang` | 0 | Secunde de hang după un apel. |
| `network_status` | inbound_outbound | Biții de stare trimiși în eter: `inbound_busy`, `inbound_idle` sau `inbound_outbound`. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Ca la DMR. |
| `assembly_ms` | 500 | Fereastra pentru asamblarea frame-urilor de voce din reflector (LDU-uri). |
| `retention_ms` | 1000 | Cât timp este păstrat un LDU complet pentru TX. Trebuie să fie mai mic decât `reflector.timeout_ms`. |
| `watchdog_ms` | 2000 | Un apel se încheie după acest interval fără trafic. |
| `call_limit_ms` | 180000 | Limita absolută a duratei unui apel. |

### 10.3 `[channel.p25.reflector]`

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `endpoint` | obligatoriu | p25.tetralink.ro:41000 | `host:port` al reflector-ului. |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62005 | Adresa UDP locală. |
| `callsign` | obligatoriu | N0CALL | 1–10 caractere, doar majuscule `A-Z 0-9 / -`. |
| `interval_ms` | 5000 | 5000 | Intervalul de poll. |
| `timeout_ms` | 15000 | 15000 | Timeout-ul legăturii. Trebuie să fie mai mare decât `interval_ms`. |
| `talkgroup` | niciunul | — | Rută fixă opțională: doar group call-urile RF către acest talkgroup merg la reflector, iar traficul primit este rescris către el. Absența înseamnă transparent. 0 este invalid. |
| `name` | din host | — | Numele afișat. |
| `password` | — | — | Nu este suportat de protocolul reflector-ului P25: setarea sa este o eroare. |
| `auto_select` | false | — | Selectează reflector-ul după talkgroup-ul apelat pe RF, ca la P25Gateway. Un group call către alt talkgroup găsit în `hosts` sau în lista de reflectoare descărcată leagă acel reflector între apeluri: over-ul care îl selectează rămâne local, următorul iese pe rețea. Necesită `talkgroup` (cel la care revine implicit). |
| `revert_secs` | 600 | — | Secunde fără trafic înainte de revenirea la `endpoint`/`talkgroup` configurat. 0 rămâne pe ultimul reflector selectat. |
| `[[…reflector.hosts]]` | niciunul | — | Intrări proprii de reflector, `talkgroup` (1–65535), `endpoint` (`host:port`) și `name` opțional; au prioritate față de lista descărcată. Până la 256. |

Exemplu cu selecție automată:

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

## 11. FM analogic

Canalul FM este repeater-ul FM MMDVM original rulând pe SDR, conectat la o rețea fie prin svxlink (RoLink), fie
printr-o legătură USRP legacy de 8 kHz. Canalul RF este de 25 kHz.

### 11.1 `[channel.fm]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `callsign` | obligatoriu | N0CALL | Callsign-ul stației (1–23 caractere tipăribile), folosit în metadatele de rețea. Textul CW ID este `keyers[0].text`. |

### 11.2 `[channel.fm.modem]`: repeater-ul FM MMDVM (toate cheile obligatorii)

| Cheie | Stație | Semnificație și reguli |
|---|---|---|
| `mode` | simplex | `link`, `simplex` sau `duplex`. |
| `access` | carrier_and_ctcss | Cum se deschide intrarea: `carrier`, `ctcss_delayed`, `carrier_and_ctcss` sau `ctcss_latch`. |
| `external_enabled` | true | Activează calea audio de rețea. |
| `rx_level` | 200 | Scala intrării RF, 256 / `rx_level`. Nu trebuie să fie 0. La 200 există rezervă; la 128 stațiile puternice erau limitate (clipping). |
| `tx_level` | 128 | Scala ieșirii. Unitățile svxlink `tx_ceiling` presupun 128. |
| `rf_boost` / `external_boost` | 1 / 1 | Multiplicatori de câștig la retransmiterea audio RF sau din rețea. |
| `ctcss_frequency` | 103 | Tonul CTCSS ca număr întreg de Hz din tabelul MMDVM (103 = 103.5 Hz). Coduri valide: 67 69 71 74 77 79 82 85 88 91 94 97 100 103 107 110 114 118 123 127 131 136 141 146 151 156 159 162 165 167 171 173 177 179 183 186 189 192 196 199 203 206 210 218 225 229 233 241 250 254. |
| `ctcss_high` / `ctcss_low` | 150 / 100 | Pragurile detectorului CTCSS, în unități MMDVM originale. |
| `ctcss_level` | 16 | Nivelul tonului CTCSS la TX: aproximativ 400 Hz deviation pe această stație. |
| `squelch_high` / `squelch_low` | 111 / 105 | Pragurile carrier squelch-ului pe metrica RSSI. Squelch-ul se deschide sau se închide după 4 citiri consecutive dincolo de un prag. Cu `cos_invert = true`, se deschide la sau sub `squelch_low` și se închide la sau peste `squelch_high`. |
| `cos_invert` | true | Inversează comparația. Pe această placă metrica măsoară zgomotul, deci un semnal puternic dă o valoare *mică*. Idle floor măsurat: 115.3. |
| `max_deviation` | 0 | Pragul de blanking la over-deviation × 128; 0 îl dezactivează. Înlocuiește audio-ul excesiv cu un bleep și liniște; **nu** este un limiter. Nu dovedește nimic despre deviation-ul RF. |
| `timeout_level` | 20 | Nivelul tonului de timeout și al bleep-ului de blanking. |
| `callsign_at_start` / `callsign_at_end` / `callsign_at_latch` | false | Când se trimite CW ID-ul. |

**`[channel.fm.modem.timers]`**: toate cheile sunt obligatorii, în milisecunde; 0 dezactivează un timer.

| Cheie | Stație | Semnificație |
|---|---|---|
| `callsign` | 600000 | Intervalul CW ID (10 min). |
| `timeout` | 180000 | Timeout-ul de TX (3 min). |
| `holdoff` | 0 | Timer-ul de hold-off. |
| `kerchunk` | 250 | Timer-ul de kerchunk. |
| `ack_min` / `ack_delay` | 1000 / 500 | Timer-ele de acknowledgement (ack). |
| `hang` | 300 | Hang time-ul repeater-ului. |

Semantica MMDVM exactă a `holdoff`, `kerchunk` și a timer-elor de acknowledgement este cea a controlerului FM MMDVM
original. Acest cod nu le documentează mai departe.

**`[[channel.fm.modem.keyers]]`**: exact trei blocuri, în această ordine: callsign, RF acknowledgement, network
acknowledgement.

| Cheie | Stație | Semnificație |
|---|---|---|
| `text` | N0CALL / K / R | Text CW, până la 255 de caractere; caracterele necunoscute sunt sărite. |
| `speed_wpm` | 20 / 18 / 22 | Viteza, nu trebuie să fie 0. |
| `frequency_hz` | 1000 / 900 / 1100 | Frecvența tonului, 1–24000. |
| `high_level` / `low_level` | 0 / 0 | Nivelurile tonului keyer-ului în unități MMDVM originale. Codul nu le documentează precis. |

### 11.3 `[channel.fm.worker]`: calea semnalului (toate cheile obligatorii, cu excepția `squelch`)

| Cheie | Stație | Semnificație |
|---|---|---|
| `symbol_deviation` | 10.0 | Scalarea modulației FM. Trebuie să fie > 0. |
| `rx_dc_block` | true | DC blocker înainte de demodulare. |
| `power_calibration` | 0 | Offset adăugat la citirea RSSI în dB. |
| `tx_peak` | 0.5 | Envelope-ul de vârf în baseband, 0 < x ≤ 0.8. |
| `host_tx_gain` | 1.0 | Câștigul audio RF → rețea. Trebuie să fie > 0. |
| `host_rx_gain` | 9.0 | Câștigul audio rețea → RF. Trebuie să fie > 0. Pe această stație svxlink trimite audio plat, nelimitat, iar nexus2 face procesarea. |
| `pre_emphasis` / `de_emphasis` | false / false | Filtre pe partea de host. Oprite aici deoarece opțiunile svxlink de mai jos fac această muncă. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Ca la DMR. |

**`[channel.fm.worker.squelch]`** (opțional; fiecare cheie are o valoare implicită):

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `mode` | fixed | fixed | `fixed`: decid pragurile modemului de mai sus. `shadow`: doar măsoară și raportează. `adaptive`: decide un noise-floor tracker. `adaptive` necesită `squelch_high = 1`, `squelch_low = 0` și `cos_invert = false`; acest lucru este verificat la pornire, nu de `doctor`. |
| `open_db` / `close_db` | 10 / 5 | — | Signal-to-noise ratio (SNR) necesar pentru deschidere și pentru menținerea deschisă (0–60; close < open). Folosit de `adaptive`. |
| `quiet_open_db` / `quiet_close_db` | 6 / 3 | — | Quieting-ul necesar pentru deschidere și pentru menținerea deschisă (0–60; close < open). |

### 11.4 `[channel.fm.svxlink]`: puntea către svxlink / RoLink

Această punte trimite PCM brut de 16 kHz peste UDP, cu squelch și PTT prin PTY-uri. Este exclusivă cu
`[channel.fm.usrp]`: configurați exact una dintre ele.

| Cheie | Implicit | Stație | Semnificație și reguli |
|---|---|---|---|
| `rx_audio` | obligatoriu | 127.0.0.1:40100 | Unde este trimis audio-ul RF. Corespunde cu svxlink `[Rx] AUDIO_DEV=udp:…`. |
| `tx_audio_bind` | obligatoriu | 127.0.0.1:40101 | Socket-ul local pentru audio-ul din rețea de la svxlink `[Tx] AUDIO_DEV=udp:…`. Trebuie să difere de `rx_audio`. |
| `ptt_pty` | obligatoriu | …/var/run/ptt | svxlink `[Tx] PTT_PTY`. |
| `squelch_pty` | obligatoriu | …/var/run/sql | svxlink `[Rx] PTY_PATH` (`SQL_DET=PTY`). Trebuie să difere de `ptt_pty`. |
| `talker_file` | niciunul | …/var/run/talker | Fișierul în care svxlink scrie talker-ul curent, afișat pe dashboard. |
| `tg_file` | niciunul | …/var/run/tg | Fișierul care conține talkgroup-ul selectat. |
| `conf` | niciunul | …/svxlink.conf | Configurația proprie a svxlink, citită o singură dată la pornire pentru numele rețelei, callsign-ul și talkgroup-urile afișate. |
| `network` | din host-ul reflector-ului | RoLink | Numele afișat. |
| `jitter_ms` | 75 | 75 | Prebuffer pentru audio-ul primit (0–500). |

**Procesarea RX (RF → rețea).** Fiecare etapă este oprită implicit; stația folosește setul recomandat.

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `rx_de_emphasis_hz` | 0 (oprit) | 300.0 | Frecvența de colț a de-emphasis-ului, 0 sau 50–2000 Hz. |
| `rx_gate_depth_db` | 0 (oprit) | 30.0 | Adâncimea downward expander-ului care urmărește noise floor-ul (0–60). Cheile de mai jos se aplică doar când aceasta este > 0. |
| `rx_gate_open_db` / `rx_gate_close_db` | 10 / 5 | — | Nivelul peste noise floor care este considerat vorbire (3–40) și cel care o menține (1…open). |
| `rx_gate_hold_ms` / `rx_gate_attack_ms` / `rx_gate_release_ms` | 120 / 5 / 60 | — | Temporizarea gate-ului (hold ≤ 2000, attack 1–200, release 5–2000). |
| `rx_floor_rise_db_per_s` | 5.0 | — | Cât de repede poate crește estimarea noise floor-ului (0.1–60). |
| `rx_level_target_dbfs` | oprit | −21.0 | Ținta leveler-ului, rms-ul vorbirii (−40…−6, sub peak ceiling). Cheile următoare se aplică doar când este setată. |
| `rx_level_initial_gain_db` / `rx_level_min_gain_db` / `rx_level_max_gain_db` | 20 / −10 / 30 | — | Câștigurile leveler-ului (min −20…max, max 0–40). |
| `rx_level_rise_db_per_s` / `rx_level_fall_db_per_s` | 8 / 30 | — | Viteza leveler-ului (0.5–60 / 1–120). |
| `rx_peak_ceiling_dbfs` | oprit | −6.0 | Plafonul look-ahead limiter-ului (−20…−0.5). |

**Procesarea TX (rețea → RF):**

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `tx_pre_emphasis` | false | true | nexus2 aplică pre-emphasis (+6 dB/octavă peste 300 Hz, unitar la 1 kHz). Setați împreună cu el svxlink `[Tx1] PREEMPHASIS=0`. |
| `tx_limiter` | false | true | Look-ahead peak limiter la `tx_ceiling`. Setați împreună cu el svxlink `LIMITER_THRESH=0`. |
| `tx_ceiling` | 2048 | 2200 | Plafonul limiter-ului (256–3072). 1 unitate ≈ 1.9 Hz deviation la `tx_level` 128, deci 2200 ≈ 4.2 kHz. |

### 11.5 `[channel.fm.usrp]`: legătura USRP legacy (alternativă la svxlink)

Toate cheile sunt obligatorii. Legătura transportă audio legacy de 8 kHz.

| Cheie | Semnificație |
|---|---|
| `bind` | `ip:port` UDP local. Nu trebuie să partajeze un port nenul cu un canal FM anterior. |
| `peer` | `ip:port` unicast de la distanță, din aceeași familie de adrese ca `bind` și diferit de acesta. |
| `loss_timeout_ms` | După acest interval fără pachete, vocea din rețea este eliberată. Trebuie să fie > 0. |

### 11.6 `[callsigns]`: identități și MDC1200

Acest tabel este folosit doar de puntea FM svxlink. Este încărcat la pornire și reconstruit cu `nexus2 callsigns`.

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `enabled` | true | true | Folosește identitățile în general. |
| `database` | niciunul | /opt/nexus-bs2/callsigns.csv | Fișier `callsign,mdc_id`. Trebuie să poată fi citit la pornire, altfel pornirea eșuează. |
| `source` | niciunul | API-ul de noduri RoLink | Lista de operatori folosită la reconstruire: un URL sau un fișier local. |
| `selection` | active | active | `active` = operatorii auziți în ultimele `active_days`; `all` = toți operatorii listați vreodată. |
| `active_days` | 30 | 60 | Fereastra de retrospectivă (1–3650). |
| `mdc_encode` | false | true | Trimite ID-ul MDC1200 al talker-ului din rețea înaintea audio-ului din rețea pe FM analogic. Doar spre exterior: MDC-ul auzit pe RF nu este niciodată considerat de încredere. |
| `unknown` | RoLink | ROLINK | Eticheta anunțată pentru un talker care lipsește din baza de date. Gol nu trimite nimic. |

## 12. Moduri amânate

Canalele `dstar`, `ysf`, `nxdn` și `pocsag` sunt interpretate și apar în `plan`/`doctor`, dar o pornire live le refuză
cât timp sunt activate. Tabelele lor acceptă `tx_delay` (toate patru), `low_deviation` și `hang_seconds` (YSF, implicit
4) și `hang_seconds` (NXDN, implicit 5). Ele există doar pentru planificare.

## 13. `[dashboard]`, `[network]`, `[touch]`

### `[dashboard]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `listen` | 127.0.0.1:8872 | 0.0.0.0:8080 | Adresa HTTP și WebSocket. O adresă non-loopback necesită `auth.enabled = true`. |
| `id_database_url` | https://radioid.net/static/user.csv | — | Sursa bazei de date callsign/nume/țară (doar https). Reîmprospătată când este mai veche de 7 zile. |
| `id_database_dir` | `<config dir>/ids` | — | Unde este stocată acea bază de date. |
| `host_lists.dmr_url` | https://www.pistar.uk/downloads/DMR_Hosts.txt | — | Lista de master-e DMR (format Pi-Star) pentru pagina Settings. |
| `host_lists.p25_url` | https://refcheck.radio/api/hostfile-gate/fetch/p25/ | — | Lista de reflectoare P25 (format P25Hosts.txt): registrul DVRef, servit de RefCheck.Radio. Folosită și de `auto_select` la P25. |
| `host_lists.p25_token` | niciunul | — | Token-ul personal RefCheck.Radio: introduceți callsign-ul pe https://hostfiles.refcheck.radio pentru a obține unul. Fără el, lista P25 nu este descărcată (intrările `hosts` definite manual funcționează în continuare). Termenii lor permit o descărcare pe oră; lista creditează DVRef. Eliminat din export-urile Share. |
| `host_lists.refresh_hours` | 24 | — | Descarcă din nou listele după acest număr de ore (1–720). Copiile din cache stau lângă baza de date de ID-uri. |
| `auth.enabled` | true | true | Autentificare HTTP Basic pe fiecare pagină, rută API și WebSocket. |
| `auth.username` | admin | setat | 1–64 caractere tipăribile, fără `:`. |
| `auth.password` | nexus | setat | 1–256 caractere. **Schimbați valoarea implicită.** |

### `[network]`

| Cheie | Implicit | Stație | Semnificație |
|---|---|---|---|
| `connect_timeout_secs` | 10 | 10 | Validat (1–300), dar în prezent nu este folosit de nicio legătură. |
| `brew` | niciunul | setat | Legătura cu core-ul TETRA, §8.5. |

### `[touch]`: panoul Nexus-BS Touch

Acest tabel este citit doar de panou, care îl recitește în 5 s de la o modificare.

| Cheie | Implicit | Semnificație |
|---|---|---|
| `backlight` | 100 | % iluminare de fundal în utilizare (1–100). |
| `screen_timeout_secs` | 300 | Secunde de inactivitate înainte de screensaver; 0 = întotdeauna pornit (maxim 86400). |
| `screensaver` | splash | `splash` = logo-ul de boot, static și estompat; `blank` = negru, cu iluminarea de fundal oprită. |
| `screensaver_dim` | 30 | % luminozitate a logo-ului estompat. |
| `screensaver_backlight` | 25 | % iluminare de fundal cât timp este afișat logo-ul. |
| `wake_on_touch` | true | O atingere trezește ecranul. Acea atingere este consumată și nu apasă niciodată un control. |
| `wake_on_traffic` | true | Un apel nou sau o intrare nouă în last-heard trezește ecranul. |

Cu un timeout setat, cel puțin una dintre cele două opțiuni de trezire trebuie să fie activă. Controlul iluminării de
fundal necesită `/sys/class/backlight` și permisiune de scriere; fără acestea panoul doar estompează imaginea.

## 14. Linia de comandă

`--config <file>` are implicit valoarea `nexus2.toml`.

| Comandă | Scop |
|---|---|
| `nexus2 plan` | Afișează planul de frecvențe fără hardware. |
| `nexus2 doctor [--live]` | Validează tot ce poate fi verificat offline. `--live` adaugă cerințele pentru pornirea live (§6). |
| `nexus2 plan-risks --occupied-half-bandwidth-hz N --rx-bandwidth-hz N --tx-bandwidth-hz N [--sample-rate N]` | Geometria offline a planului privind DC, image și intermodulation. |
| `nexus2 init [--defaults]` | Creează un fișier nou (TX inhibat). Refuză să suprascrie unul existent. |
| `nexus2 callsigns --out FILE [...]` | Reconstruiește baza de date callsign/MDC1200 din `[callsigns]`. |
| `nexus2 run --live [--allow-rf-tx]` | Stația propriu-zisă (serviciul systemd). |
| `nexus2 manage ...` | API-ul de management HTTPS (serviciu separat). |

## 15. Rețete

**Mutarea unui carrier.** Modificați-i `offset_hz` cu un multiplu de 25 kHz. Păstrați 50 kHz față de vecinii săi și
rămâneți în interiorul ±(sample rate / 2 − 20 kHz). Rulați `nexus2 plan`, apoi `doctor --live` (TETRA trebuie să
păstreze o codificare standard), aplicați și reporniți.

**Scoaterea definitivă a unui mod din eter.** Setați `enabled = false` pe `[[channel]]`-ul său. Își păstrează slotul
de frecvență. Pentru DMR, P25 sau FM o oprire temporară se poate face live din dashboard și durează până la
următoarea repornire.

**Activarea carrier-ului de packet data TETRA.** Setați `enabled = true` pe `tetra-data`, păstrându-i identitatea
egală cu a primarului, și reporniți. Primarul își păstrează `[channel.tetra.packet_data]`.

**Ajustarea nivelului TX al unui mod.** Modificați `tx_peak` al acelui mod: `[channel.tetra.worker]`,
`[channel.dmr.worker]`, `[channel.p25.worker]` sau `[channel.fm.worker]`. Păstrați-l la 0.8 sau mai puțin. Totalul
tuturor carrier-elor împarte un singur DAC, așa că creșterea unui mod crește composite peak-ul. `sdr.tx_gain_db` și
câștigurile pe elemente mută **toate** modurile împreună.

**Măsurarea nivelurilor RX.** Setați `sdr.io_diagnostics = true`, reporniți, colectați liniile de jurnal `RX_LEVEL`,
apoi setați-l înapoi la `false`.

**Schimbarea parolei dashboard-ului.** Setați `dashboard.auth.password` și reporniți serviciul radio. Panoul touch se
autentifică în dashboard cu credențialele din același fișier, dar le citește o singură dată, la pornire, așa că
reporniți și `nexus-panel.service`.

## 16. Limitări cunoscute și puncte deschise

Acestea provin din codul în starea lui actuală; sunt raportate pentru ca nimeni să nu se bazeze pe ele:

- `network.connect_timeout_secs` este validat, dar nicio legătură nu îl folosește.
- Limita de 0.8 pentru `tx_peak` TETRA, documentată în cod, nu este impusă; este impus doar 0–1.0, când TX este
  activat.
- `packet_data.max_slots` în afara 1–4 este readus fără avertisment în intervalul 1–4, nu respins.
- `sdr.driver` nu selectează dispozitivul (îl selectează `uri`). Un nume greșit de driver schimbă doar comportamentul
  specific SX1255.
- `nexus2 callsigns` revine fără avertisment la valorile implicite dacă fișierul nu se încarcă și poate interoga SDR-ul când
  `[sdr]` folosește auto.
- Nedocumentate în acest cod: unitățile exacte ale `receiver_delay` DMR; ce anume declanșează `stall_ms`; scalele
  `high_level`/`low_level` ale keyer-ului FM și scalele pragurilor/nivelului CTCSS (unități MMDVM originale, pe un
  octet); cel mai lung text CW care încape.
- Reîncărcarea setărilor per canal fără repornire există în cod, dar nu are niciun apelant în producție. Doar
  comutatorul pornit/oprit pentru DMR, P25 și FM funcționează live.
