# Manual de configuración de Nexus-BS 2.0 (`nexus2.toml`)

Versión en inglés: [Manual_conf.md](Manual_conf.md)

Versión en rumano: [Manual_conf.ro.md](Manual_conf.ro.md)

Este manual explica cada clave del archivo de configuración de la estación, cómo se comprueba el archivo
y cómo llegan los cambios a la estación en funcionamiento. Se redactó a partir del
código fuente (commit `d7fdec7` y posteriores), no de memoria. Cuando el código no
resuelve un detalle, el manual lo dice en lugar de suponerlo.

La configuración de la estación en funcionamiento es la referencia para los valores recomendados. El único
ejemplo incluido con Nexus-BS 2.0, `examples/nexus2.example.toml`, usa los mismos ajustes con identidades
genéricas (`N0CALL`, `CHANGE_ME`), el plan de frecuencias del ejemplo TETRA BlueStation y TX bloqueada. En las tablas siguientes:

- **Predeterminado** es lo que usa el código cuando la clave no está presente. "obligatorio" significa que no hay valor predeterminado.
- **Estación** es el valor en el archivo de la estación en funcionamiento. Un guion significa que la clave no está presente allí.

Contenido:
1. [Cómo se lee el archivo](#1-cómo-se-lee-el-archivo)
2. [Aplicar un cambio](#2-aplicar-un-cambio)
3. [Frecuencias y plan de canales](#3-frecuencias-y-plan-de-canales)
4. [Permiso de TX y el interruptor de RF](#4-permiso-de-tx-y-el-interruptor-de-rf)
5. [`[sdr]`](#5-sdr-la-radio-compartida)
6. [`[[channel]]`](#6-channel-claves-comunes)
7. [`[channel.access]`](#7-channelaccess-política-de-admisión)
8. [TETRA](#8-tetra)
9. [DMR](#9-dmr)
10. [P25](#10-p25)
11. [FM analógica](#11-fm-analógica)
12. [Modos aplazados](#12-modos-aplazados)
13. [`[dashboard]`, `[network]`, `[touch]`](#13-dashboard-network-touch)
14. [Línea de comandos](#14-línea-de-comandos)
15. [Recetas](#15-recetas)
16. [Limitaciones conocidas y puntos abiertos](#16-limitaciones-conocidas-y-puntos-abiertos)

---

## 1. Cómo se lee el archivo

- **Formato.** El archivo es TOML 1.0. Dentro de una tabla, el orden de las claves es libre. Un encabezado de tabla como
  `[network.brew]` puede aparecer en cualquier lugar del archivo, incluso después de las entradas `[[channel]]`. El archivo de la estación aprovecha
  esto para agrupar cada enlace de red con su modo.
- **Estricto.** Todas las tablas rechazan claves desconocidas. Un error tipográfico es un error fatal, por ejemplo
  `config: unknown field `brightnes` near line 70`. Los mensajes de error indican la ruta de la clave y el número de línea, nunca el
  valor, para que los secretos no se filtren a los registros.
- **Tablas obligatorias.** `[sdr]`, al menos un `[[channel]]`, `[dashboard]` y `[network]` son obligatorias.
  `[dashboard]` y `[network]` pueden estar vacías. `[touch]` y `[callsigns]` son opcionales.
- **Formas de los valores.**
  - Se acepta un entero donde se espera un decimal (`rx_gain_db = 30`).
  - Los literales hexadecimales, octales y binarios funcionan para cualquier entero (`nac = 0x293`), y también los guiones bajos (`offset_hz = -50_000`).
  - Los números negativos deben escribirse en decimal.
- **Unidades.** Las frecuencias están en Hz. Los tiempos están en ms salvo que el nombre de la clave termine en `_secs`.
- **Secretos.** Los secretos son cadenas de texto simples: `dashboard.auth.password`, `network.brew.password`,
  `channel.dmr.network.password` y `sdr.uri`. Se ocultan en la salida de depuración y en la exportación depurada.
  Mantenga el archivo con modo 0600. El dashboard y la API de gestión lo escriben ambos de esa forma.

## 2. Aplicar un cambio

**El proceso de radio lee el archivo una sola vez, al arrancar.** Editar el archivo no cambia nada hasta que el servicio
se reinicia. Hay dos excepciones:

| Qué | Cuándo surte efecto |
|---|---|
| `[touch]` | El panel lo vuelve a leer en unos 5 s. Un valor fuera de rango vuelve a su valor predeterminado. |
| Habilitar o deshabilitar un canal **DMR, P25 o FM** desde el dashboard o el conmutador del panel | Inmediatamente, sin reinicio. Se mantiene **solo en memoria** y no se escribe en el archivo: tras un reinicio del servicio vuelve a aplicarse el valor `enabled` del archivo. TETRA no tiene conmutador en vivo. |

Todo lo demás necesita un reinicio. `[callsigns]` también se carga solo al arrancar.

**Comprobar antes de aplicar:**

```sh
nexus2 --config nexus2.toml plan         # frequency plan, no hardware
nexus2 --config nexus2.toml doctor       # full static validation
nexus2 --config nexus2.toml doctor --live   # plus the checks a live start makes
```

`doctor` nunca abre el SDR ni la red. No resuelve DNS, no enlaza sockets ni lee la base de datos de `[callsigns]`,
por lo que un `doctor` limpio no garantiza que los enlaces de red se establezcan.

**En la estación** (API de gestión desde una estación de trabajo; véase MANAGEMENT-API.md):

```sh
A="python3 scripts/nexus2-admin.py --url https://<station-ip>:9443 --connect-ip <station-ip>"
$A config > current.json                 # contains secrets: keep private
$A validate candidate.toml               # validate with the station's own checks
$A set-config candidate.toml --if-match <config_sha256 from status>
```

- `set-config` sustituye el archivo, conserva una copia de seguridad, reinicia la radio y revierte el cambio si el proceso no se mantiene
  en buen estado.
- La precondición de hash se niega a sobrescribir un archivo que haya cambiado entretanto.
- El reinicio interrumpe brevemente **todos los modos**.
- El gestor valida con su propio binario. Tras una actualización de la radio que añada claves nuevas, actualice también el gestor;
  de lo contrario las rechaza (HTTP 422).
- Mientras la API de gestión esté instalada (archivo marcador `/opt/nexus-bs2/.management-enabled`), el editor de
  configuración propio del dashboard y su formulario de ajustes se niegan a guardar y responden `use_management_api`.

## 3. Frecuencias y plan de canales

Un único SDR lleva todos los carriers. Cada canal es un offset respecto a los dos local oscillators (LO):

```
RX frequency (uplink, terminals -> station)   = sdr.rx_lo_hz + offset_hz
TX frequency (downlink, station -> terminals) = sdr.tx_lo_hz + offset_hz
```

Hay un único `offset_hz` por canal, de modo que todos los carriers tienen el mismo duplex split: `tx_lo_hz - rx_lo_hz`.

**Reglas del plan.** Se comprueban para **todos** los canales, incluidos los deshabilitados:

| Regla | Error si no se cumple |
|---|---|
| De 1 a 7 canales en total | número de canales |
| `offset_hz` es un múltiplo exacto de 25 000 | `channel[i].offset_hz: must be an exact multiple of 25000` |
| Dos canales cualesquiera separados al menos 50 000 Hz (una celda vacía de 25 kHz entre carriers) | error de plan |
| `abs(offset_hz) + 20 000 < sample_rate / 2` | "insufficient Nyquist/filter margin" |
| Placas SX1255 (`sx`, `mucell`): ningún canal **habilitado** en `offset_hz = 0`, porque quedaría sobre DC | "carrier would sit on DC; shift rx_lo_hz/tx_lo_hz by a 25 kHz multiple instead" |

- El passband analógico necesario es `2 × (max |offset_hz| + 20 000)`. Para la estación es 2 × (175 000 + 20 000)
  = 390 kHz.
- **Sample rate.** Cuando `sdr.sample_rate` no está presente o vale 0, la estación pregunta a la radio qué sample rates admite. Conserva
  los que son como máximo 1 MHz, múltiplo de 50 kHz y más anchos que el passband necesario. Después toma el menor
  de los que sean al menos el doble del passband; si no hay ninguno, el más ancho utilizable. La estación termina en 600 kS/s.
- **Codificación de frecuencias TETRA.** Un carrier TETRA debe ser además un canal TETRA estándar:
  - La frecuencia de TX debe estar en el raster de 25 kHz, opcionalmente desplazada +6.25, −6.25 o +12.5 kHz.
  - El spacing RX–TX debe ser uno de los duplex spacings estándar de la banda. En la banda de 400 MHz son 10, 7, 8,
    5 o 9.5 MHz. La estación usa 7 MHz.
  - En caso contrario, el error es `tetra: frequencies have no supported standard repeater encoding`.
  - `plan` no comprueba esto; `doctor` y un arranque en vivo sí.

**Plan de la estación** (LO 431.2875 / 438.2875 MHz):

| Canal | offset_hz | RX MHz | TX MHz | Estado |
|---|---|---|---|---|
| tetra-data | −50 000 | 431.2375 | 438.2375 | deshabilitado |
| dmr | +25 000 | 431.3125 | 438.3125 | habilitado |
| tetra | +75 000 | 431.3625 | 438.3625 | habilitado |
| p25 | +125 000 | 431.4125 | 438.4125 | habilitado |
| analog (FM) | +175 000 | 431.4625 | 438.4625 | habilitado |

Para mover toda la estación, cambie ambos LO en la misma cantidad. Para mover un carrier, cambie su `offset_hz`, respete las
reglas del grid y compruebe con `nexus2 plan`.

## 4. Permiso de TX y el interruptor de RF

Tres cosas independientes deciden si la estación transmite:

1. **`sdr.tx_inhibited`** (configuración, valor predeterminado en el código `true`). Cuando es true, no se prepara ninguna ruta de TX. El archivo
   de la estación establece `false`. `nexus2 init` siempre escribe `true`.
2. **`--allow-rf-tx`** (línea de comandos de `run --live`). Es solo un permiso del proceso. Un archivo con
   `tx_inhibited = false` se rechaza al arrancar si falta. La unidad del servicio lo pasa.
3. **El interruptor global de RF** (RF KILL / RESTORE RF en el dashboard y en el panel). **No es una clave de configuración**.
   - Su estado se guarda en `~/.local/state/nexus2/rf-<hash of the config path>`, por lo que sobrevive a los reinicios. OFF detiene la RX
     y la TX de todos los modos.
   - Solo un RF KILL del operador, o un fallo fatal al detener la radio de forma limpia, guarda OFF.
   - Un arranque fallido mantiene ON y reintenta por sí solo, a los 2 s y luego con una pausa creciente de hasta 60 s. Un ejemplo es un
     nombre de red que todavía no se resuelve durante el arranque del sistema.
   - Es un control por software, no un enclavamiento de hardware.

## 5. `[sdr]`: la radio compartida

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `driver` | `"auto"` | — | Nombre del driver de SoapySDR, o `auto`/vacío para usar la mejor radio encontrada. **No** se usa para abrir el dispositivo: para eso se usa `uri`. Solo indica a la estación si la placa es una SX1255 (`sx`, `mucell`). Eso afecta a la regla de DC, al valor predeterminado de `tx_lead_quanta` y a una advertencia sobre el HAT instalado. Orden de preferencia en auto: placas SX1255 (`sx` antes que `mucell`), Lime, USRP B2xx, otros USRP, Pluto, cualquier otra. |
| `uri` | `"auto"` | — | Argumentos de dispositivo de SoapySDR (1–256 caracteres), o `auto` para tomarlos de la radio encontrada. |
| `sample_rate` | `0` (auto) | — | Sample rate complejo de todo el dispositivo. Debe ser un múltiplo positivo de 50 000 y como máximo 1 000 000. Véase §3. |
| `rx_lo_hz` | obligatorio | 431287500 | Local oscillator de RX. |
| `tx_lo_hz` | obligatorio | 438287500 | Local oscillator de TX. |
| `rx_gain_db` | obligatorio | 30.0 | Ganancia global de RX (−100…100), aplicada antes de las ganancias por elemento. |
| `tx_gain_db` | obligatorio | 0.0 | Ganancia global de TX (−100…100). |
| `rx_gains_db.<element>` | ninguno | LNA 42, PGA 16 | Ganancias de RX por elemento, aplicadas después de la ganancia global. Los nombres de los elementos los da el driver. |
| `tx_gains_db.<element>` | ninguno | DAC 9, MIXER 30 | Ganancias de TX por elemento. |
| `rx_bandwidth_hz` / `tx_bandwidth_hz` | auto | — | Ancho del filtro analógico. Si se define, debe ser **mayor que** el passband necesario (§3). Si no está presente se usa el rango del driver. |
| `rx_channel` / `tx_channel` | 0 | — | Índice del canal de hardware en radios multicanal. |
| `rx_antenna` / `tx_antenna` | según la placa | — | Puerto de antena. Los valores predeterminados son SX1255 `RX`/`TX`, Lime USB `LNAL`/`BAND1`, otros Lime `LNAW`/`BAND2`, Pluto `A_BALANCED`/`A`, USRP `TX/RX`. |
| `rx_stream_args` / `tx_stream_args` | vacío | — | Argumentos adicionales de stream de SoapySDR (`key = "value"`). |
| `settings.<key>` | vacío | — | Ajustes de SoapySDR para todo el dispositivo. Las placas SX1255 reciben `PA = "AUTO"` salvo que se defina aquí. |
| `ppm` | 0.0 | — | Corrección de frecuencia para ambas rutas (−100…100). Un valor distinto de cero requiere un driver que admita corrección; de lo contrario la radio no se abre. |
| `tx_lead_quanta` | 8 en SX1255, 3 en las demás | — | Cuántos bloques de 1 ms de muestras de TX se preparan por adelantado respecto al reloj de la radio (1–8). 8 está medido en la SX1255; 3 no está medido en radios USB. Súbalo primero si el registro muestra "TX late block skipped". |
| `sx1255_rx_pll_bw_khz` | driver (300) | — | Ancho de banda del lazo del PLL de RX de la SX1255: 75, 150, 225 o 300. Se rechaza en otras radios. |
| `sx1255_tx_pll_bw_khz` | driver (150) | 300 | Ancho de banda del lazo del PLL de TX de la SX1255. 300 da el menor close-in phase noise en el carrier. |
| `tx_inhibited` | **true** | false | Véase §4. |
| `io_diagnostics` | false | false | Ayuda de medida: ventanas de nivel de RX y contadores de temporización de E/S, además del bloque `rf_io` del dashboard. Escribe unas 11 líneas por segundo en el journal, así que déjelo desactivado en funcionamiento normal. No altera la RF. |
| `pipeline_diagnostics` | false | — | Instrumentación experimental de colas y canales. |

## 6. `[[channel]]`: claves comunes

Cada bloque `[[channel]]` es un carrier. Su tabla de modo debe seguirlo, junto con `[channel.access]` para
DMR, P25 y FM.

| Clave | Predeterminado | Significado y reglas |
|---|---|---|
| `id` | obligatorio | Nombre único, 1–64 caracteres de `A-Z a-z 0-9 - _`. El dashboard, la API de control y un `cell` de packet data se refieren a los canales por él. |
| `mode` | obligatorio | `tetra`, `dmr`, `p25` o `fm`. `dstar`, `ysf`, `nxdn` y `pocsag` también se analizan, pero un arranque en vivo los rechaza si están habilitados (§12). La tabla correspondiente (`[channel.tetra]`, …) es obligatoria y cualquier otra tabla de modo es un error. |
| `enabled` | true | Encendido/apagado administrativo. Un canal deshabilitado sigue contando para las reglas del plan y para el límite de 7 carriers. |
| `offset_hz` | obligatorio | Offset respecto a ambos LO (§3). |

**Orden de los bloques `[[channel]]`.** El orden no cambia ninguna frecuencia. Determina el índice con el que se muestra un canal
en los mensajes de error (`channel[2].…`) y en el formulario de ajustes del dashboard.

- El nivel de TX de TETRA se toma del primer canal TETRA habilitado que tenga un `[channel.tetra.worker]`
  (`tx_peak`). Mantenga primero el canal TETRA primario.
- Los conflictos de puertos USRP se comprueban solo contra canales anteriores.
- Reordenar, añadir o quitar canales necesita un reinicio.

**Requisitos del arranque en vivo.** Van más allá de lo que comprueba `doctor` sin `--live`:

- Exactamente un canal TETRA primario habilitado, para que la estación no funcione sin TETRA.
- Todo otro canal TETRA habilitado debe ser el carrier de packet data de esa celda.
- Un canal DMR habilitado necesita `worker`, `network` y `network.runtime`.
- Un canal P25 habilitado necesita `worker` y `reflector`.
- Un canal FM habilitado necesita `modem`, `worker` y además `svxlink` o `usrp`.
- Todo canal DMR, P25 o FM habilitado necesita `[channel.access]`.

## 7. `[channel.access]`: política de admisión

Esta tabla decide qué tráfico deja pasar un modo. Por defecto no se admite nada y nunca concede TX física.
`mode` debe ser igual al modo del canal. TETRA no tiene tabla de acceso.

| Modo | Clave | Significado |
|---|---|---|
| dmr | `rf_slots = [TS1, TS2]` (obligatorio) | Admitir tráfico de RF (voz, datos, CSBK) en cada timeslot. |
| dmr | `network_slots = [TS1, TS2]` (obligatorio) | Admitir tráfico de red hacia RF en cada timeslot. |
| dmr | `protected_voice` (predeterminado false) | Admitir voz que lleve el indicador de privacidad/cifrado (PI). Cuando es false se rechaza en ambos sentidos. |
| p25 | `rf` (obligatorio) | Admitir llamadas de RF. |
| p25 | `network` (obligatorio) | Admitir tráfico del reflector hacia RF. |
| fm | `selected` (obligatorio) | La entrada "FM mode selected" de MMDVM de la lógica del repeater. No es el acceso por carrier/CTCSS. |

## 8. TETRA

### 8.1 `[channel.tetra]`

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `mcc` | obligatorio | 901 | MCC, mobile country code (0–1023). |
| `mnc` | obligatorio | 9999 | MNC, mobile network code (0–16383). |
| `location_area` | obligatorio | 2 | Location area base (0–16383). |
| `colour_code` | obligatorio | 1 | Colour code (0–63). |
| `rotate_location_area_on_start` | true | — | Cuando es true, cada arranque difunde un LA distinto derivado de `location_area`, de modo que los terminales acampados se vuelvan a registrar tras un reinicio. Póngalo en false para difundir exactamente `location_area`. |
| `implicit_group_affiliation` | true | — | Una llamada o floor request de un terminal que no se sabe que esté adscrito a ese group lo adscribe en lugar de rechazarse. |
| `station_name` | `"Nexus-BS 2.0"` | — | Texto del mensaje periódico Home Mode Display que se muestra en los terminales. Vacío lo desactiva. |
| `timezone` | zona horaria del host | — | Nombre de zona IANA (p. ej. `Europe/Bucharest`) para la difusión del reloj de la celda. Vacío desactiva el reloj. Un nombre no válido hace fallar `doctor`. |
| `role` | `"primary"` | — / `packet_data` | `packet_data` convierte este canal en un carrier de datos secundario (§8.4). |
| `cell` | ninguno | — / `tetra` | Solo para `role = "packet_data"`: el `id` del canal primario. |

### 8.2 `[channel.tetra.packet_data]`: IP sobre TETRA (solo primario)

Cuando está presente, la celda ofrece packet data SNDCP. Los terminales obtienen direcciones del pool y llegan a la estación a través de
una interfaz tun de Linux.

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `ipv4_pool` | obligatorio | `10.44.0.0/24` | CIDR con prefijo /16…/30 y sin bits de host. El primer host es la pasarela en la interfaz tun (10.44.0.1); el resto se asigna a los terminales. |
| `tun` | `"tetra0"` | — | Nombre de la interfaz tun (1–15 caracteres). |
| `mtu` | 576 | — | Tamaño de N-PDU y MTU de la interfaz (128–1500). |
| `header_compression` | true | false | Conceder compresión de cabeceras RFC 1144 (Van Jacobson) cuando un terminal la solicite. El Motorola MXP600 abandona la activación cuando se concede, por lo que la estación la mantiene desactivada. |
| `max_slots` | 4 | — | Máximo de timeslots que un terminal puede usar en el carrier de datos. Los valores fuera de 1–4 se **limitan silenciosamente**, no se rechazan. |
| `advertise` | true | — | Anunciar "SNDCP available" en la información del sistema. Si está desactivado, el servicio sigue respondiendo a los terminales que lo intenten. |
| `advertise_advanced_link` | true | — | Anunciar "advanced link supported". |

### 8.3 `[channel.tetra.worker]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `tx_peak` | 0.5 | 0.8 | Envolvente de pico en banda base del carrier TETRA, en la misma escala que el `tx_peak` de DMR, P25 y FM. 0.5 da el mismo pico que DMR; aproximadamente 0.79 da la misma potencia media, con picos 4 dB más altos. Manténgalo en 0.8 o menos. El código solo rechaza valores fuera de 0–1.0, y solo cuando la TX está habilitada. |

### 8.4 Carrier de packet data (`role = "packet_data"`)

Un segundo carrier TETRA de la misma celda, con packet data en los cuatro timeslots y sin canal de control. Los terminales
son enviados a él mediante asignación de canal desde el primario. Reglas:

- `cell` debe nombrar un canal TETRA primario existente, y ese primario debe tener `[channel.tetra.packet_data]`.
- `mcc`, `mnc`, `location_area` y `colour_code` deben ser iguales a los del primario.
- Un carrier de packet data no tiene tabla `packet_data` propia, y hay como máximo uno por primario.
- Sus frecuencias deben estar en la banda del primario con el mismo duplex spacing.
- Cuando está deshabilitado (`enabled = false`), la celda mantiene el packet data solo en el carrier primario.

### 8.5 `[network.brew]`: enlace con el núcleo TETRA (Brew / TetraPack)

Esta tabla requiere al menos un canal TETRA habilitado.

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `host` | obligatorio | core.tetrapack.online | Nombre de host o IP del servidor. |
| `port` | 443 | 443 | Puerto del servidor. |
| `tls` | true | true | Usar TLS (wss/https). |
| `username` / `password` | ninguno | definido | Inicio de sesión HTTP Digest. O ambos o ninguno. |
| `reconnect_delay_secs` | 15 | 15 | Pausa antes de volver a conectar (1–300). |
| `jitter_initial_latency_frames` | 0 | — | Delay inicial de reproducción adicional, en tramas, para la voz de red. |
| `feature_sds_enabled` | true | — | Pasar mensajes SDS entre terminales locales y de red. |
| `feature_rssi_export` | false | — | Informar el RSSI al servidor. |
| `whitelisted_ssis` | ninguno | — | Permitir solo llamadas de red con estos SSI remotos (hasta 4096 entradas, cada una como máximo 0xFFFFFF). |

Las claves heredadas `network.brew_endpoint` y `network.brew_token` todavía se analizan, pero `doctor` y un arranque en vivo
las rechazan. Use `[network.brew]` en su lugar.

## 9. DMR

### 9.1 `[channel.dmr]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `id` | obligatorio | 1234567 | ID de repeater de 24 bits en RF (1–16777215). También es el ID de red salvo que se defina `network.id`. |
| `colour_code` | obligatorio | 1 | Colour code (0–15). |

### 9.2 `[channel.dmr.worker]`: módem nativo (todas las claves son obligatorias)

| Clave | Estación | Significado y reglas |
|---|---|---|
| `symbol_deviation` | 10.0 | Escalado FM de los símbolos 4FSK, usado para el acondicionamiento de RX y para TX. Debe ser > 0. |
| `tx_level_q15` | −13056 | Ganancia de deviation de TX en Q15 con signo. −13056 = −102 × 128, el nivel de referencia del SDR. Un valor negativo invierte la deviation. |
| `tx_peak` | 0.5 | Envolvente de pico en banda base, 0 < x ≤ 0.8. |
| `power_calibration` | 0 | Offset que se suma a la lectura de RSSI en dB. |
| `receiver_delay` | 3 | Delay por timeslot del receptor del repeater. Su unidad exacta no está documentada en este código fuente; mantenga 3. |
| `marker_offset_numerator` / `marker_offset_denominator` | 720 / 1 | Corrección entre el marcador de temporización de TX y la RX acondicionada, en muestras a 24 kS/s: 720 = 30 ms. El denominador no debe ser 0. |
| `maximum_input_age_ms` | 100 | Entrada de RX más antigua que el worker todavía acepta. Debe ser > 0. |
| `stall_ms` | 5000 | Timeout por bloqueo (stall) del worker. Debe ser > 0. |
| `hang_frames` | 51 | Hang frames enviados tras un terminador de llamada (0–60). |

### 9.3 `[channel.dmr.network]`: enlace MMDVM homebrew

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `profile` | obligatorio | brandmeister | `brandmeister`, `dmrplus`, `tgif`, `freedmr` o `custom` (un master propio). Todos hablan el protocolo MMDVM homebrew; solo el perfil BrandMeister añade las comprobaciones de `version`/`software` indicadas más abajo. La página Settings ofrece los masters de la red elegida a partir de la lista descargada (véase `[dashboard.host_lists]`). |
| `endpoint` | obligatorio | 2262.master.brandmeister.network:62031 | `host:port` del master, resuelto cuando arranca el canal. |
| `password` | obligatorio | definido | Contraseña de inicio de sesión (1–256 bytes). |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62002 | Dirección UDP local. El puerto 0 deja que el sistema elija. |
| `callsign` | obligatorio | N0CALL | 1–8 caracteres. |
| `id` | `[channel.dmr] id` | 123456701 | ID de red. El ID efectivo debe ser > 1000. |
| `name` | del host | — | Nombre mostrado en el dashboard. |
| `retry_ms` | 5000 | 1000 | Intervalo de reintento del inicio de sesión. Debe ser > 0. |
| `timeout_ms` | 15000 | 60000 | Timeout de la sesión. Debe ser mayor que `retry_ms`. |

### 9.4 `[channel.dmr.network.runtime]`: datos de la estación y política de sesión

Todas las claves son obligatorias salvo `rf_inactivity_timeout_ms`.

| Clave | Estación | Significado y reglas |
|---|---|---|
| `power`, `latitude`, `longitude`, `height` | 0, 0.0, 0.0, 0 | Datos de la estación enviados al iniciar sesión: potencia 0–99, latitud ±90, longitud ±180, altura 0–999. |
| `location` | "" | Hasta 20 caracteres (el texto más largo se recorta). |
| `description` | "Nexus-BS 2.0 hotspot" | Hasta 19 caracteres (el texto más largo se recorta). |
| `url` | "" | Hasta 124 caracteres. |
| `version` | 20260916_Nexus | Con BrandMeister debe ser `YYYYMMDD_…`. |
| `software` | MMDVM_Nexus | Con BrandMeister debe empezar por `MMDVM`; de lo contrario el master rechaza el inicio de sesión. |
| `options` | "" | Cadena de opciones enviada al master, p. ej. `TS1=1;TS2=1`. |
| `retention_ms` | 250 | Retención de la voz de red. Debe ser > 0. |
| `rf_inactivity_timeout_ms` | 5000 | Timeout por inactividad de RF. Omítalo o ponga un valor > 0. |
| `embedded_lc_only` | true | true = enviar solo el link control embebido en la voz de red; false = conservar los datos embebidos entrantes. |

## 10. P25

### 10.1 `[channel.p25]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `nac` | obligatorio | 0x293 | NAC, Network Access Code, 0x000–0xFFF. Escríbalo en hexadecimal tal como está programado en las radios. El dashboard lo muestra en decimal (659). |

### 10.2 `[channel.p25.worker]`: módem nativo (todas las claves son obligatorias)

| Clave | Estación | Significado y reglas |
|---|---|---|
| `symbol_deviation`, `tx_level_q15`, `tx_peak` | 10.0, −13056, 0.5 | Como en DMR (§9.2). |
| `duplex` | true | Transmisor duplex. `tx_hang` solo funciona en duplex. |
| `tx_delay` | 0 | Unidades originales de MMDVM: 500 ms + valor × 10 ms, con un máximo de 1 s. |
| `tx_hang` | 0 | Segundos de hang tras una llamada. |
| `network_status` | inbound_outbound | Bits de estado enviados al aire: `inbound_busy`, `inbound_idle` o `inbound_outbound`. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Como en DMR. |
| `assembly_ms` | 500 | Ventana para ensamblar las tramas de voz del reflector (LDU). |
| `retention_ms` | 1000 | Cuánto tiempo se retiene un LDU completo para su transmisión. Debe ser menor que `reflector.timeout_ms`. |
| `watchdog_ms` | 2000 | Una llamada termina tras este tiempo sin tráfico. |
| `call_limit_ms` | 180000 | Límite absoluto de duración de la llamada. |

### 10.3 `[channel.p25.reflector]`

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `endpoint` | obligatorio | p25.tetralink.ro:41000 | `host:port` del reflector. |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62005 | Dirección UDP local. |
| `callsign` | obligatorio | N0CALL | 1–10 caracteres, solo `A-Z 0-9 / -` en mayúsculas. |
| `interval_ms` | 5000 | 5000 | Intervalo de poll. |
| `timeout_ms` | 15000 | 15000 | Timeout del enlace. Debe ser mayor que `interval_ms`. |
| `talkgroup` | ninguno | — | Ruta fija opcional: solo las group calls de RF a este talkgroup van al reflector, y el tráfico entrante se reescribe a él. Si no está presente, el enlace es transparente. 0 no es válido. |
| `name` | del host | — | Nombre mostrado. |
| `password` | — | — | No lo admite el protocolo del reflector P25: definirlo es un error. |
| `auto_select` | false | — | Selecciona el reflector según el talkgroup marcado en RF, como hace P25Gateway. Una group call a otro talkgroup encontrado en `hosts` o en la lista de reflectores descargada enlaza ese reflector entre llamadas: el over que lo selecciona queda local, el siguiente sale a la red. Requiere `talkgroup` (el valor por defecto al que vuelve). |
| `revert_secs` | 600 | — | Segundos sin tráfico antes de volver al `endpoint`/`talkgroup` configurado. 0 se queda en el último reflector seleccionado. |
| `[[…reflector.hosts]]` | ninguno | — | Entradas de reflector propias, `talkgroup` (1–65535), `endpoint` (`host:port`) y `name` opcional; tienen prioridad sobre la lista descargada. Hasta 256. |

Ejemplo con selección automática:

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

## 11. FM analógica

El canal FM es el repeater FM original de MMDVM ejecutándose sobre el SDR, enlazado a una red a través de svxlink
(RoLink) o de un enlace USRP heredado de 8 kHz. El canal de RF es de 25 kHz.

### 11.1 `[channel.fm]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `callsign` | obligatorio | N0CALL | Callsign de la estación (1–23 caracteres imprimibles), usado en los metadatos de red. El texto del CW ID es `keyers[0].text`. |

### 11.2 `[channel.fm.modem]`: repeater FM de MMDVM (todas las claves son obligatorias)

| Clave | Estación | Significado y reglas |
|---|---|---|
| `mode` | simplex | `link`, `simplex` o `duplex`. |
| `access` | carrier_and_ctcss | Cómo se abre la entrada: `carrier`, `ctcss_delayed`, `carrier_and_ctcss` o `ctcss_latch`. |
| `external_enabled` | true | Habilita la ruta de audio de red. |
| `rx_level` | 200 | Escala de la entrada de RF, 256 / `rx_level`. No debe ser 0. Con 200 hay margen; con 128 se recortaban las estaciones fuertes. |
| `tx_level` | 128 | Escala de salida. Las unidades de `tx_ceiling` de svxlink presuponen 128. |
| `rf_boost` / `external_boost` | 1 / 1 | Multiplicadores de ganancia al retransmitir audio de RF o de red. |
| `ctcss_frequency` | 103 | Tono CTCSS como Hz enteros de la tabla de MMDVM (103 = 103.5 Hz). Códigos válidos: 67 69 71 74 77 79 82 85 88 91 94 97 100 103 107 110 114 118 123 127 131 136 141 146 151 156 159 162 165 167 171 173 177 179 183 186 189 192 196 199 203 206 210 218 225 229 233 241 250 254. |
| `ctcss_high` / `ctcss_low` | 150 / 100 | Umbrales del detector CTCSS, en unidades originales de MMDVM. |
| `ctcss_level` | 16 | Nivel del tono CTCSS en TX: unos 400 Hz de deviation en esta estación. |
| `squelch_high` / `squelch_low` | 111 / 105 | Umbrales del carrier squelch sobre la métrica RSSI. El squelch se abre o se cierra tras 4 lecturas consecutivas más allá de un umbral. Con `cos_invert = true`, se abre en `squelch_low` o por debajo y se cierra en `squelch_high` o por encima. |
| `cos_invert` | true | Invierte la comparación. En esta placa la métrica mide ruido, de modo que una señal fuerte da un valor *bajo*. Suelo en reposo medido: 115.3. |
| `max_deviation` | 0 | Umbral de blanking por over-deviation × 128; 0 lo desactiva. Sustituye el audio excesivo por un pitido y silencio; **no** es un limiter. No demuestra nada sobre la deviation de RF. |
| `timeout_level` | 20 | Nivel del tono de timeout y del pitido de blanking. |
| `callsign_at_start` / `callsign_at_end` / `callsign_at_latch` | false | Cuándo enviar el CW ID. |

**`[channel.fm.modem.timers]`**: todas las claves son obligatorias, en milisegundos; 0 desactiva un temporizador.

| Clave | Estación | Significado |
|---|---|---|
| `callsign` | 600000 | Intervalo del CW ID (10 min). |
| `timeout` | 180000 | Timeout de TX (3 min). |
| `holdoff` | 0 | Temporizador de hold-off. |
| `kerchunk` | 250 | Temporizador de kerchunk. |
| `ack_min` / `ack_delay` | 1000 / 500 | Temporizadores de acknowledgement. |
| `hang` | 300 | Hang time del repeater. |

La semántica exacta de MMDVM de `holdoff`, `kerchunk` y los temporizadores de acknowledgement es la del controlador FM original de
MMDVM. Este código fuente no la documenta más.

**`[[channel.fm.modem.keyers]]`**: exactamente tres bloques, en este orden: callsign, acknowledgement de RF, acknowledgement
de red.

| Clave | Estación | Significado |
|---|---|---|
| `text` | N0CALL / K / R | Texto CW, hasta 255 caracteres; los caracteres desconocidos se omiten. |
| `speed_wpm` | 20 / 18 / 22 | Velocidad, no debe ser 0. |
| `frequency_hz` | 1000 / 900 / 1100 | Frecuencia del tono, 1–24000. |
| `high_level` / `low_level` | 0 / 0 | Niveles del tono del keyer en unidades originales de MMDVM. El código fuente no los documenta con precisión. |

### 11.3 `[channel.fm.worker]`: ruta de la señal (todas las claves son obligatorias salvo `squelch`)

| Clave | Estación | Significado |
|---|---|---|
| `symbol_deviation` | 10.0 | Escalado de la modulación FM. Debe ser > 0. |
| `rx_dc_block` | true | Bloqueador de DC antes de la demodulación. |
| `power_calibration` | 0 | Offset que se suma a la lectura de RSSI en dB. |
| `tx_peak` | 0.5 | Envolvente de pico en banda base, 0 < x ≤ 0.8. |
| `host_tx_gain` | 1.0 | Ganancia de audio RF → red. Debe ser > 0. |
| `host_rx_gain` | 9.0 | Ganancia de audio red → RF. Debe ser > 0. En esta estación svxlink envía audio plano y sin limitar, y nexus2 hace el procesado. |
| `pre_emphasis` / `de_emphasis` | false / false | Filtros del lado del host. Desactivados aquí porque las opciones de svxlink de más abajo hacen ese trabajo. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Como en DMR. |

**`[channel.fm.worker.squelch]`** (opcional; todas las claves tienen valor predeterminado):

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `mode` | fixed | fixed | `fixed`: deciden los umbrales del módem indicados arriba. `shadow`: solo medir e informar. `adaptive`: decide un seguidor del noise floor. `adaptive` requiere `squelch_high = 1`, `squelch_low = 0` y `cos_invert = false`; esto se comprueba al arrancar, no en `doctor`. |
| `open_db` / `close_db` | 10 / 5 | — | SNR (signal-to-noise ratio) necesaria para abrir y para mantenerse abierto (0–60; close < open). La usa `adaptive`. |
| `quiet_open_db` / `quiet_close_db` | 6 / 3 | — | Quieting necesario para abrir y para mantenerse abierto (0–60; close < open). |

### 11.4 `[channel.fm.svxlink]`: puente hacia svxlink / RoLink

Este puente envía PCM en bruto de 16 kHz por UDP, con squelch y PTT a través de PTY. Es excluyente con `[channel.fm.usrp]`:
configure exactamente uno.

| Clave | Predeterminado | Estación | Significado y reglas |
|---|---|---|---|
| `rx_audio` | obligatorio | 127.0.0.1:40100 | Adónde se envía el audio de RF. Corresponde a `[Rx] AUDIO_DEV=udp:…` de svxlink. |
| `tx_audio_bind` | obligatorio | 127.0.0.1:40101 | Socket local para el audio de red procedente de `[Tx] AUDIO_DEV=udp:…` de svxlink. Debe ser distinto de `rx_audio`. |
| `ptt_pty` | obligatorio | …/var/run/ptt | `[Tx] PTT_PTY` de svxlink. |
| `squelch_pty` | obligatorio | …/var/run/sql | `[Rx] PTY_PATH` de svxlink (`SQL_DET=PTY`). Debe ser distinto de `ptt_pty`. |
| `talker_file` | ninguno | …/var/run/talker | Archivo donde svxlink escribe el talker actual, que se muestra en el dashboard. |
| `tg_file` | ninguno | …/var/run/tg | Archivo que contiene el talkgroup seleccionado. |
| `conf` | ninguno | …/svxlink.conf | Configuración propia de svxlink, leída una vez al arrancar para el nombre de red, el callsign y los talkgroups mostrados. |
| `network` | del host del reflector | RoLink | Nombre mostrado. |
| `jitter_ms` | 75 | 75 | Prebuffer del audio entrante (0–500). |

**Procesado de RX (RF → red).** Todas las etapas están desactivadas por defecto; la estación usa el conjunto recomendado.

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `rx_de_emphasis_hz` | 0 (desactivado) | 300.0 | Frecuencia de corte del de-emphasis, 0 o 50–2000 Hz. |
| `rx_gate_depth_db` | 0 (desactivado) | 30.0 | Profundidad del downward expander que sigue el noise floor (0–60). Las claves siguientes solo se aplican cuando es > 0. |
| `rx_gate_open_db` / `rx_gate_close_db` | 10 / 5 | — | Nivel por encima del floor que cuenta como voz (3–40) y que la mantiene (1…open). |
| `rx_gate_hold_ms` / `rx_gate_attack_ms` / `rx_gate_release_ms` | 120 / 5 / 60 | — | Temporización del gate (hold ≤ 2000, attack 1–200, release 5–2000). |
| `rx_floor_rise_db_per_s` | 5.0 | — | Con qué rapidez puede subir la estimación del floor (0.1–60). |
| `rx_level_target_dbfs` | desactivado | −21.0 | Objetivo del leveler, rms de la voz (−40…−6, por debajo del peak ceiling). Las claves siguientes solo se aplican cuando está definido. |
| `rx_level_initial_gain_db` / `rx_level_min_gain_db` / `rx_level_max_gain_db` | 20 / −10 / 30 | — | Ganancias del leveler (min −20…max, max 0–40). |
| `rx_level_rise_db_per_s` / `rx_level_fall_db_per_s` | 8 / 30 | — | Velocidad del leveler (0.5–60 / 1–120). |
| `rx_peak_ceiling_dbfs` | desactivado | −6.0 | Ceiling del look-ahead limiter (−20…−0.5). |

**Procesado de TX (red → RF):**

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `tx_pre_emphasis` | false | true | nexus2 aplica pre-emphasis (+6 dB/octava por encima de 300 Hz, ganancia unidad a 1 kHz). Configure junto con ello `[Tx1] PREEMPHASIS=0` en svxlink. |
| `tx_limiter` | false | true | Look-ahead peak limiter en `tx_ceiling`. Configure junto con ello `LIMITER_THRESH=0` en svxlink. |
| `tx_ceiling` | 2048 | 2200 | Ceiling del limiter (256–3072). 1 unidad ≈ 1.9 Hz de deviation con `tx_level` 128, por lo que 2200 ≈ 4.2 kHz. |

### 11.5 `[channel.fm.usrp]`: enlace USRP heredado (alternativa a svxlink)

Todas las claves son obligatorias. El enlace transporta audio heredado de 8 kHz.

| Clave | Significado |
|---|---|
| `bind` | `ip:port` UDP local. No debe compartir un puerto distinto de cero con un canal FM anterior. |
| `peer` | `ip:port` unicast remoto, de la misma familia de direcciones que `bind` y distinto de ella. |
| `loss_timeout_ms` | Tras este tiempo sin paquetes, se libera la voz de red. Debe ser > 0. |

### 11.6 `[callsigns]`: identidades y MDC1200

Esta tabla la usa solo el puente FM de svxlink. Se carga al arrancar y se reconstruye con `nexus2 callsigns`.

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `enabled` | true | true | Usar las identidades en absoluto. |
| `database` | ninguno | /opt/nexus-bs2/callsigns.csv | Archivo `callsign,mdc_id`. Debe poder leerse al arrancar; de lo contrario el arranque falla. |
| `source` | ninguno | API de nodos de RoLink | Lista de operadores usada al reconstruir: una URL o un archivo local. |
| `selection` | active | active | `active` = operadores escuchados en los últimos `active_days`; `all` = todos los operadores que hayan figurado alguna vez en la lista. |
| `active_days` | 30 | 60 | Ventana retrospectiva (1–3650). |
| `mdc_encode` | false | true | Enviar el ID MDC1200 del talker de red antes del audio de red en FM analógica. Solo hacia fuera: nunca se confía en el MDC escuchado en RF. |
| `unknown` | RoLink | ROLINK | Etiqueta anunciada para un talker que no figura en la base de datos. Vacía no envía nada. |

## 12. Modos aplazados

Los canales `dstar`, `ysf`, `nxdn` y `pocsag` se analizan y aparecen en `plan`/`doctor`, pero un arranque en vivo los rechaza mientras
estén habilitados. Sus tablas aceptan `tx_delay` (los cuatro), `low_deviation` y `hang_seconds` (YSF, predeterminado 4) y
`hang_seconds` (NXDN, predeterminado 5). Existen solo para la planificación.

## 13. `[dashboard]`, `[network]`, `[touch]`

### `[dashboard]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `listen` | 127.0.0.1:8872 | 0.0.0.0:8080 | Dirección HTTP y WebSocket. Una dirección que no sea de loopback requiere `auth.enabled = true`. |
| `id_database_url` | https://radioid.net/static/user.csv | — | Origen de la base de datos de callsign/nombre/país (solo https). Se actualiza cuando tiene más de 7 días. |
| `id_database_dir` | `<config dir>/ids` | — | Dónde se guarda esa base de datos. |
| `host_lists.dmr_url` | https://www.pistar.uk/downloads/DMR_Hosts.txt | — | Lista de masters DMR (formato Pi-Star) para la página Settings. |
| `host_lists.p25_url` | https://refcheck.radio/api/hostfile-gate/fetch/p25/ | — | Lista de reflectores P25 (formato P25Hosts.txt): el registro DVRef, servido por RefCheck.Radio. También la usa `auto_select` en P25. |
| `host_lists.p25_token` | ninguno | — | Su token personal de RefCheck.Radio: introduzca su callsign en https://hostfiles.refcheck.radio para obtener uno. Sin él, la lista P25 no se descarga (sus `hosts` definidos manualmente siguen funcionando). Sus condiciones permiten una descarga por hora; la lista acredita a DVRef. Se elimina de las exportaciones Share. |
| `host_lists.refresh_hours` | 24 | — | Vuelve a descargar las listas tras este número de horas (1–720). Las copias en caché se guardan junto a la base de datos de ID. |
| `auth.enabled` | true | true | Autenticación HTTP Basic en todas las páginas, rutas de la API y el WebSocket. |
| `auth.username` | admin | definido | 1–64 caracteres imprimibles, sin `:`. |
| `auth.password` | nexus | definido | 1–256 caracteres. **Cambie el valor predeterminado.** |

### `[network]`

| Clave | Predeterminado | Estación | Significado |
|---|---|---|---|
| `connect_timeout_secs` | 10 | 10 | Se valida (1–300) pero actualmente ningún enlace lo usa. |
| `brew` | ninguno | definido | Enlace con el núcleo TETRA, §8.5. |

### `[touch]`: panel Nexus-BS Touch

Esta tabla la lee solo el panel, que la vuelve a leer en los 5 s siguientes a un cambio.

| Clave | Predeterminado | Significado |
|---|---|---|
| `backlight` | 100 | % de retroiluminación en uso (1–100). |
| `screen_timeout_secs` | 300 | Segundos de inactividad antes del salvapantallas; 0 = siempre encendida (máximo 86400). |
| `screensaver` | splash | `splash` = el logotipo de arranque, estático y atenuado; `blank` = negro con la retroiluminación apagada. |
| `screensaver_dim` | 30 | % de brillo del logotipo atenuado. |
| `screensaver_backlight` | 25 | % de retroiluminación mientras se muestra el logotipo. |
| `wake_on_touch` | true | Un toque despierta la pantalla. Ese toque se consume y nunca pulsa un control. |
| `wake_on_traffic` | true | Una llamada nueva o una nueva entrada de last-heard despierta la pantalla. |

Con un tiempo de espera definido, al menos una de las dos opciones de despertar debe estar activada. El control de la retroiluminación necesita
`/sys/class/backlight` y permiso de escritura; sin ellos el panel solo atenúa la imagen.

## 14. Línea de comandos

`--config <file>` tiene como valor predeterminado `nexus2.toml`.

| Comando | Finalidad |
|---|---|
| `nexus2 plan` | Mostrar el plan de frecuencias sin hardware. |
| `nexus2 doctor [--live]` | Validar todo lo que puede comprobarse sin conexión. `--live` añade los requisitos del arranque en vivo (§6). |
| `nexus2 plan-risks --occupied-half-bandwidth-hz N --rx-bandwidth-hz N --tx-bandwidth-hz N [--sample-rate N]` | Geometría de DC, imagen e intermodulación del plan, sin conexión. |
| `nexus2 init [--defaults]` | Crear un archivo nuevo (TX inhibida). Se niega a sobrescribir uno existente. |
| `nexus2 callsigns --out FILE [...]` | Reconstruir la base de datos de callsigns/MDC1200 a partir de `[callsigns]`. |
| `nexus2 run --live [--allow-rf-tx]` | La estación propiamente dicha (el servicio systemd). |
| `nexus2 manage ...` | La API de gestión HTTPS (servicio aparte). |

## 15. Recetas

**Mover un carrier.** Cambie su `offset_hz` en un múltiplo de 25 kHz. Mantenga 50 kHz respecto a sus vecinos y quédese dentro de
±(sample rate / 2 − 20 kHz). Ejecute `nexus2 plan`, luego `doctor --live` (TETRA debe conservar una codificación estándar), aplique y
reinicie.

**Sacar un modo del aire de forma permanente.** Ponga `enabled = false` en su `[[channel]]`. Conserva su posición de frecuencia. Para
DMR, P25 o FM, una desactivación temporal puede hacerse en vivo desde el dashboard, y dura hasta el siguiente reinicio.

**Habilitar el carrier TETRA de packet data.** Ponga `enabled = true` en `tetra-data`, manteniendo su identidad igual a la del
primario, y reinicie. El primario conserva su `[channel.tetra.packet_data]`.

**Ajustar el nivel de TX de un modo.** Cambie el `tx_peak` de ese modo: `[channel.tetra.worker]`, `[channel.dmr.worker]`,
`[channel.p25.worker]` o `[channel.fm.worker]`. Manténgalo en 0.8 o menos. El total de todos los carriers comparte un único DAC, así que
subir un modo sube el pico compuesto. `sdr.tx_gain_db` y las ganancias por elemento mueven **todos** los modos a la vez.

**Medir los niveles de RX.** Ponga `sdr.io_diagnostics = true`, reinicie, recoja las líneas `RX_LEVEL` del journal y después vuelva a
ponerlo en `false`.

**Cambiar la contraseña del dashboard.** Defina `dashboard.auth.password` y reinicie el servicio de radio. El panel táctil
inicia sesión en el dashboard con las credenciales de este mismo archivo, pero solo las lee una vez al arrancar, así que
reinicie también `nexus-panel.service`.

## 16. Limitaciones conocidas y puntos abiertos

Proceden del código en su estado actual; se indican para que nadie confíe en ellas:

- `network.connect_timeout_secs` se valida pero ningún enlace lo usa.
- El límite de 0.8 para el `tx_peak` de TETRA documentado en el código no se aplica; solo se aplica 0–1.0, cuando la TX está habilitada.
- `packet_data.max_slots` fuera de 1–4 se limita silenciosamente.
- `sdr.driver` no selecciona el dispositivo (lo hace `uri`). Un nombre de driver erróneo solo cambia el comportamiento
  específico de la SX1255.
- `nexus2 callsigns` recurre discretamente a los valores predeterminados si el archivo no se carga, y puede sondear el SDR cuando `[sdr]`
  usa auto.
- No documentado en este código fuente: las unidades exactas de `receiver_delay` de DMR; qué dispara exactamente `stall_ms`; las escalas de
  `high_level`/`low_level` del keyer FM y de umbral/nivel de CTCSS (unidades de byte originales de MMDVM); el texto CW más largo que
  cabe.
- La recarga de ajustes por canal sin reinicio existe en el código pero no tiene ningún llamador en producción. Solo el conmutador de
  encendido/apagado de DMR, P25 y FM funciona en vivo.
