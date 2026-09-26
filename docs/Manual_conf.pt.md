# Manual de configuração do Nexus-BS 2.0 (`nexus2.toml`)

Versão em inglês: [Manual_conf.md](Manual_conf.md)

Este manual explica cada chave do ficheiro de configuração da estação, como o ficheiro
é verificado e como as alterações chegam à estação em funcionamento. Foi escrito a partir do
código-fonte (commit `d7fdec7` e posteriores), não de memória. Quando o código não
esclarece um detalhe, o manual di-lo em vez de adivinhar.

A configuração da estação em funcionamento é a referência para os valores recomendados. O único
exemplo fornecido com o Nexus-BS 2.0, `examples/nexus2.example.toml`, usa as mesmas definições com identidades
genéricas (`N0CALL`, `CHANGE_ME`), o plano de frequências do exemplo TETRA BlueStation e TX bloqueado. Nas tabelas abaixo:

- **Predefinido** é o que o código usa quando a chave está ausente. “obrigatório” significa que não há valor predefinido.
- **Estação** é o valor no ficheiro da estação em funcionamento. Um travessão significa que a chave está ausente nesse ficheiro.

Índice:
1. [Como o ficheiro é lido](#1-como-o-ficheiro-é-lido)
2. [Aplicar uma alteração](#2-aplicar-uma-alteração)
3. [Frequências e o plano de canais](#3-frequências-e-o-plano-de-canais)
4. [Permissão de TX e o comutador de RF](#4-permissão-de-tx-e-o-comutador-de-rf)
5. [`[sdr]`](#5-sdr-o-rádio-partilhado)
6. [`[[channel]]`](#6-channel-chaves-comuns)
7. [`[channel.access]`](#7-channelaccess-política-de-admissão)
8. [TETRA](#8-tetra)
9. [DMR](#9-dmr)
10. [P25](#10-p25)
11. [FM analógico](#11-fm-analógico)
12. [Modos adiados](#12-modos-adiados)
13. [`[dashboard]`, `[network]`, `[touch]`](#13-dashboard-network-touch)
14. [Linha de comandos](#14-linha-de-comandos)
15. [Receitas](#15-receitas)
16. [Limitações conhecidas e pontos em aberto](#16-limitações-conhecidas-e-pontos-em-aberto)

---

## 1. Como o ficheiro é lido

- **Formato.** O ficheiro é TOML 1.0. Dentro de uma tabela, a ordem das chaves é livre. Um cabeçalho de tabela como
  `[network.brew]` pode aparecer em qualquer ponto do ficheiro, mesmo depois das entradas `[[channel]]`. O ficheiro da
  estação usa isto para agrupar cada ligação de rede com o respetivo modo.
- **Estrito.** Todas as tabelas rejeitam chaves desconhecidas. Um erro de digitação é um erro fatal, por exemplo
  `config: unknown field `brightnes` near line 70`. As mensagens de erro indicam o caminho da chave e o número da linha, nunca o
  valor, para que os segredos não passem para os registos (logs).
- **Tabelas obrigatórias.** `[sdr]`, pelo menos um `[[channel]]`, `[dashboard]` e `[network]` são obrigatórias.
  `[dashboard]` e `[network]` podem estar vazias. `[touch]` e `[callsigns]` são opcionais.
- **Formas dos valores.**
  - Um inteiro é aceite onde se espera um decimal (`rx_gain_db = 30`).
  - Literais hexadecimais, octais e binários funcionam para qualquer inteiro (`nac = 0x293`), tal como os sublinhados (`offset_hz = -50_000`).
  - Os números negativos têm de ser escritos em decimal.
- **Unidades.** As frequências são em Hz. Os tempos são em ms, a menos que o nome da chave termine em `_secs`.
- **Segredos.** Os segredos são strings simples: `dashboard.auth.password`, `network.brew.password`,
  `channel.dmr.network.password` e `sdr.uri`. São ocultados na saída de depuração e na exportação redigida.
  Mantenha o ficheiro com o modo 0600. O dashboard e a API de gestão escrevem-no ambos dessa forma.

## 2. Aplicar uma alteração

**O processo de rádio lê o ficheiro uma única vez, ao arrancar.** Editar o ficheiro não altera nada até o serviço
ser reiniciado. Há duas exceções:

| O quê | Entra em vigor |
|---|---|
| `[touch]` | O painel volta a lê-lo em cerca de 5 s. Um valor fora do intervalo reverte para o seu valor predefinido. |
| Ativar ou desativar um canal **DMR, P25 ou FM** a partir do comutador do dashboard ou do painel | Imediatamente, sem reinício. É mantido **apenas em memória** e não é escrito no ficheiro: após um reinício do serviço volta a aplicar-se o valor `enabled` do ficheiro. O TETRA não tem comutação em funcionamento. |

Tudo o resto exige um reinício. `[callsigns]` também só é carregada no arranque.

**Verificar antes de aplicar:**

```sh
nexus2 --config nexus2.toml plan         # frequency plan, no hardware
nexus2 --config nexus2.toml doctor       # full static validation
nexus2 --config nexus2.toml doctor --live   # plus the checks a live start makes
```

O `doctor` nunca abre o SDR nem a rede. Não resolve DNS, não associa sockets nem lê a base de dados `[callsigns]`,
pelo que um `doctor` sem erros não garante que as ligações de rede fiquem ativas.

**Na estação** (API de gestão a partir de uma estação de trabalho; ver MANAGEMENT-API.md):

```sh
A="python3 scripts/nexus2-admin.py --url https://<station-ip>:9443 --connect-ip <station-ip>"
$A config > current.json                 # contains secrets: keep private
$A validate candidate.toml               # validate with the station's own checks
$A set-config candidate.toml --if-match <config_sha256 from status>
```

- `set-config` substitui o ficheiro, guarda uma cópia de segurança, reinicia o rádio e reverte se o processo não se
  mantiver saudável.
- A pré-condição do hash recusa sobrescrever um ficheiro que tenha sido alterado entretanto.
- O reinício interrompe brevemente **todos os modos**.
- O gestor valida com o seu próprio binário. Após uma atualização do rádio que acrescente chaves novas, atualize também o gestor;
  caso contrário, ele rejeita-as (HTTP 422).
- Enquanto a API de gestão estiver instalada (ficheiro marcador `/opt/nexus-bs2/.management-enabled`), o editor de
  configuração e o formulário de definições do próprio dashboard recusam guardar e respondem `use_management_api`.

## 3. Frequências e o plano de canais

Um único SDR transporta todas as carriers. Cada canal é um offset relativamente aos dois LOs (local oscillators):

```
RX frequency (uplink, terminals -> station)   = sdr.rx_lo_hz + offset_hz
TX frequency (downlink, station -> terminals) = sdr.tx_lo_hz + offset_hz
```

Há um único `offset_hz` por canal, pelo que todas as carriers têm o mesmo duplex split: `tx_lo_hz - rx_lo_hz`.

**Regras do plano.** São verificadas para **todos** os canais, incluindo os desativados:

| Regra | Erro se violada |
|---|---|
| 1 a 7 canais no total | número de canais |
| `offset_hz` é um múltiplo exato de 25 000 | `channel[i].offset_hz: must be an exact multiple of 25000` |
| Quaisquer dois canais afastados pelo menos 50 000 Hz (uma célula vazia de 25 kHz entre carriers) | erro de plano |
| `abs(offset_hz) + 20 000 < sample_rate / 2` | "insufficient Nyquist/filter margin" |
| Placas SX1255 (`sx`, `mucell`): nenhum canal **ativado** em `offset_hz = 0`, porque ficaria sobre o DC | "carrier would sit on DC; shift rx_lo_hz/tx_lo_hz by a 25 kHz multiple instead" |

- A passband analógica necessária é `2 × (max |offset_hz| + 20 000)`. Para a estação, isso dá 2 × (175 000 + 20 000)
  = 390 kHz.
- **Sample rate.** Quando `sdr.sample_rate` está ausente ou é 0, a estação pergunta ao rádio que sample rates suporta. Mantém
  os que são no máximo 1 MHz, múltiplos de 50 kHz e mais largos do que a passband necessária. Depois escolhe o menor
  que seja pelo menos o dobro da passband; caso contrário, o mais largo utilizável. A estação acaba em 600 kS/s.
- **Codificação de frequência TETRA.** Uma carrier TETRA tem também de ser um canal TETRA normalizado:
  - A frequência de TX tem de estar na grelha de 25 kHz, opcionalmente deslocada de +6.25, −6.25 ou +12.5 kHz.
  - O duplex spacing RX–TX tem de ser um dos duplex spacings normalizados para a banda. Na banda dos 400 MHz são 10, 7, 8,
    5 ou 9.5 MHz. A estação usa 7 MHz.
  - Caso contrário, o erro é `tetra: frequencies have no supported standard repeater encoding`.
  - O `plan` não verifica isto; o `doctor` e um arranque em funcionamento (live) verificam.

**Plano da estação** (LOs 431.2875 / 438.2875 MHz):

| Canal | offset_hz | RX MHz | TX MHz | Estado |
|---|---|---|---|---|
| tetra-data | −50 000 | 431.2375 | 438.2375 | desativado |
| dmr | +25 000 | 431.3125 | 438.3125 | ativado |
| tetra | +75 000 | 431.3625 | 438.3625 | ativado |
| p25 | +125 000 | 431.4125 | 438.4125 | ativado |
| analog (FM) | +175 000 | 431.4625 | 438.4625 | ativado |

Para mover a estação inteira, altere ambos os LOs no mesmo valor. Para mover uma carrier, altere o seu `offset_hz`, respeite as
regras da grelha e verifique com `nexus2 plan`.

## 4. Permissão de TX e o comutador de RF

Três elementos distintos decidem se a estação transmite:

1. **`sdr.tx_inhibited`** (configuração, predefinido no código `true`). Quando é true, nenhum caminho de TX é preparado. O ficheiro
   da estação define `false`. `nexus2 init` escreve sempre `true`.
2. **`--allow-rf-tx`** (linha de comandos de `run --live`). É apenas uma permissão do processo. Um ficheiro com
   `tx_inhibited = false` é recusado no arranque sem ela. A unidade de serviço passa-a.
3. **O comutador global de RF** (RF KILL / RESTORE RF no dashboard e no painel). **Não é uma chave de configuração**.
   - O seu estado é guardado em `~/.local/state/nexus2/rf-<hash of the config path>`, pelo que sobrevive a reinícios. OFF para o RX
     e o TX de todos os modos.
   - Só um RF KILL do operador, ou uma falha fatal ao parar o rádio de forma limpa, guarda OFF.
   - Um arranque falhado mantém ON e tenta de novo por si próprio, após 2 s e depois com uma pausa crescente até 60 s. Um exemplo é um
     nome de rede que ainda não é resolvido durante o arranque do sistema.
   - Trata-se de um controlo por software, não de um bloqueio (interlock) de hardware.

## 5. `[sdr]`: o rádio partilhado

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `driver` | `"auto"` | — | Nome do driver SoapySDR, ou `auto`/vazio para usar o melhor rádio encontrado. **Não** é usado para abrir o dispositivo: quem o faz é `uri`. Serve apenas para indicar à estação se a placa é uma SX1255 (`sx`, `mucell`). Isso afeta a regra do DC, o `tx_lead_quanta` predefinido e um aviso sobre o HAT instalado. Ordem de preferência automática: placas SX1255 (`sx` antes de `mucell`), Lime, USRP B2xx, outros USRP, Pluto, qualquer outro. |
| `uri` | `"auto"` | — | Argumentos de dispositivo SoapySDR (1–256 caracteres), ou `auto` para os obter do rádio encontrado. |
| `sample_rate` | `0` (auto) | — | Sample rate complexo de todo o dispositivo. Tem de ser um múltiplo positivo de 50 000 e no máximo 1 000 000. Ver §3. |
| `rx_lo_hz` | obrigatório | 431287500 | Local oscillator (LO) de RX. |
| `tx_lo_hz` | obrigatório | 438287500 | Local oscillator (LO) de TX. |
| `rx_gain_db` | obrigatório | 30.0 | Ganho global de RX (−100…100), aplicado antes dos ganhos por elemento. |
| `tx_gain_db` | obrigatório | 0.0 | Ganho global de TX (−100…100). |
| `rx_gains_db.<element>` | nenhum | LNA 42, PGA 16 | Ganhos de RX por elemento, aplicados depois do ganho global. Os nomes dos elementos vêm do driver. |
| `tx_gains_db.<element>` | nenhum | DAC 9, MIXER 30 | Ganhos de TX por elemento. |
| `rx_bandwidth_hz` / `tx_bandwidth_hz` | automático | — | Largura do filtro analógico. Se for definida, tem de ser **maior do que** a passband necessária (§3). Quando ausente, é usada a gama do driver. |
| `rx_channel` / `tx_channel` | 0 | — | Índice do canal de hardware em rádios multicanal. |
| `rx_antenna` / `tx_antenna` | conforme a placa | — | Porta de antena. Os predefinidos são SX1255 `RX`/`TX`, Lime USB `LNAL`/`BAND1`, outros Lime `LNAW`/`BAND2`, Pluto `A_BALANCED`/`A`, USRP `TX/RX`. |
| `rx_stream_args` / `tx_stream_args` | vazio | — | Argumentos adicionais de stream SoapySDR (`key = "value"`). |
| `settings.<key>` | vazio | — | Definições SoapySDR de todo o dispositivo. As placas SX1255 recebem `PA = "AUTO"`, a menos que seja definido aqui. |
| `ppm` | 0.0 | — | Correção de frequência para ambos os caminhos (−100…100). Um valor diferente de zero exige um driver que suporte correção; caso contrário, o rádio não abre. |
| `tx_lead_quanta` | 8 em SX1255, 3 nos restantes | — | Quantos blocos de 1 ms de amostras de TX são preparados antecipadamente em relação ao relógio do rádio (1–8). O valor 8 foi medido no SX1255; o 3 não foi medido em rádios USB. Aumente-o primeiro se o registo mostrar "TX late block skipped". |
| `sx1255_rx_pll_bw_khz` | driver (300) | — | Loop bandwidth da PLL de RX do SX1255: 75, 150, 225 ou 300. Rejeitada noutros rádios. |
| `sx1255_tx_pll_bw_khz` | driver (150) | 300 | Loop bandwidth da PLL de TX do SX1255. 300 dá o menor close-in phase noise na carrier. |
| `tx_inhibited` | **true** | false | Ver §4. |
| `io_diagnostics` | false | false | Auxiliar de medição: janelas de nível de RX e contadores de temporização de E/S, mais o bloco `rf_io` do dashboard. Escreve cerca de 11 linhas de journal por segundo, por isso mantenha-o desligado em funcionamento normal. Não altera a RF. |
| `pipeline_diagnostics` | false | — | Instrumentação experimental de filas e canais. |

## 6. `[[channel]]`: chaves comuns

Cada bloco `[[channel]]` é uma carrier. A tabela do respetivo modo tem de vir a seguir, juntamente com `[channel.access]` para
DMR, P25 e FM.

| Chave | Predefinido | Significado e regras |
|---|---|---|
| `id` | obrigatório | Nome único, 1–64 caracteres de `A-Z a-z 0-9 - _`. O dashboard, a API de controlo e um `cell` de packet data referem-se aos canais por ele. |
| `mode` | obrigatório | `tetra`, `dmr`, `p25` ou `fm`. `dstar`, `ysf`, `nxdn` e `pocsag` também são interpretados, mas um arranque em funcionamento recusa-os se estiverem ativados (§12). A tabela correspondente (`[channel.tetra]`, …) é obrigatória e qualquer outra tabela de modo é um erro. |
| `enabled` | true | Ligar/desligar administrativo. Um canal desativado continua a contar para as regras do plano e para o limite de 7 carriers. |
| `offset_hz` | obrigatório | Offset relativamente a ambos os LOs (§3). |

**Ordem dos blocos `[[channel]]`.** A ordem não altera nenhuma frequência. Define o índice com que um canal é apresentado
nas mensagens de erro (`channel[2].…`) e no formulário de definições do dashboard.

- O nível de TX TETRA é obtido do primeiro canal TETRA ativado que tenha um `[channel.tetra.worker]`
  (`tx_peak`). Mantenha o canal TETRA primário em primeiro lugar.
- Os conflitos de portas USRP são verificados apenas em relação aos canais anteriores.
- Reordenar, acrescentar ou remover canais exige um reinício.

**Requisitos do arranque em funcionamento (live).** Vão além do que o `doctor` verifica sem `--live`:

- Exatamente um canal TETRA primário ativado, para que a estação não funcione sem TETRA.
- Qualquer outro canal TETRA ativado tem de ser a carrier de packet data dessa célula.
- Um canal DMR ativado precisa de `worker`, `network` e `network.runtime`.
- Um canal P25 ativado precisa de `worker` e `reflector`.
- Um canal FM ativado precisa de `modem`, `worker` e de `svxlink` ou `usrp`.
- Todos os canais DMR, P25 ou FM ativados precisam de `[channel.access]`.

## 7. `[channel.access]`: política de admissão

Esta tabela decide que tráfego um modo deixa passar. Por predefinição nada é admitido, e ela nunca concede o TX físico.
`mode` tem de ser igual ao modo do canal. O TETRA não tem tabela de acesso.

| Modo | Chave | Significado |
|---|---|---|
| dmr | `rf_slots = [TS1, TS2]` (obrigatório) | Admitir tráfego de RF (voz, dados, CSBK) em cada timeslot. |
| dmr | `network_slots = [TS1, TS2]` (obrigatório) | Admitir tráfego de rede em direção à RF em cada timeslot. |
| dmr | `protected_voice` (predefinido false) | Admitir voz com o indicador de privacidade/cifra (PI). Quando é false, é recusada em ambos os sentidos. |
| p25 | `rf` (obrigatório) | Admitir chamadas de RF. |
| p25 | `network` (obrigatório) | Admitir tráfego do reflector em direção à RF. |
| fm | `selected` (obrigatório) | A entrada MMDVM “FM mode selected” da lógica do repeater. Não é o acesso por carrier/CTCSS. |

## 8. TETRA

### 8.1 `[channel.tetra]`

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `mcc` | obrigatório | 901 | Mobile country code (0–1023). |
| `mnc` | obrigatório | 9999 | Mobile network code (0–16383). |
| `location_area` | obrigatório | 2 | Location area base (0–16383). |
| `colour_code` | obrigatório | 1 | Colour code (0–63). |
| `rotate_location_area_on_start` | true | — | Quando é true, cada arranque difunde uma LA diferente, derivada de `location_area`, para que os terminais camped voltem a registar-se após um reinício. Defina false para difundir exatamente `location_area`. |
| `implicit_group_affiliation` | true | — | Uma chamada ou floor request de um terminal que não se sabe estar afiliado a esse group afilia-o em vez de ser recusada. |
| `station_name` | `"Nexus-BS 2.0"` | — | Texto da mensagem periódica Home Mode Display apresentada nos terminais. Vazio desliga-a. |
| `timezone` | fuso horário do host | — | Nome de zona IANA (p. ex. `Europe/Bucharest`) para a difusão do relógio da célula. Vazio desliga o relógio. Um nome inválido faz falhar o `doctor`. |
| `role` | `"primary"` | — / `packet_data` | `packet_data` torna este canal uma carrier de dados secundária (§8.4). |
| `cell` | nenhum | — / `tetra` | Apenas para `role = "packet_data"`: o `id` do canal primário. |

### 8.2 `[channel.tetra.packet_data]`: IP sobre TETRA (apenas primário)

Quando presente, a célula oferece packet data SNDCP. Os terminais obtêm endereços do pool e chegam à estação através
de uma interface tun Linux.

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `ipv4_pool` | obrigatório | `10.44.0.0/24` | CIDR com prefixo /16…/30 e sem bits de host. O primeiro host é o gateway na interface tun (10.44.0.1); os restantes são atribuídos aos terminais. |
| `tun` | `"tetra0"` | — | Nome da interface tun (1–15 caracteres). |
| `mtu` | 576 | — | Tamanho da N-PDU e MTU da interface (128–1500). |
| `header_compression` | true | false | Conceder compressão de cabeçalhos RFC 1144 (Van Jacobson) quando um terminal a pede. O Motorola MXP600 abandona a ativação quando ela é concedida, por isso a estação mantém-na desligada. |
| `max_slots` | 4 | — | Número máximo de timeslots que um terminal pode usar na carrier de dados. Valores fora de 1–4 são **limitados silenciosamente**, não rejeitados. |
| `advertise` | true | — | Anunciar “SNDCP available” na informação de sistema. Quando desligado, o serviço continua a responder aos terminais que tentem. |
| `advertise_advanced_link` | true | — | Anunciar “advanced link supported”. |

### 8.3 `[channel.tetra.worker]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `tx_peak` | 0.5 | 0.8 | Envolvente de pico em banda base da carrier TETRA, na mesma escala do `tx_peak` de DMR, P25 e FM. 0.5 dá o mesmo pico que o DMR; cerca de 0.79 dá a mesma potência média, com picos 4 dB mais altos. Mantenha-o em 0.8 ou abaixo. O código só rejeita valores fora de 0–1.0, e apenas quando o TX está ativado. |

### 8.4 Carrier de packet data (`role = "packet_data"`)

Uma segunda carrier TETRA da mesma célula, com packet data nos quatro timeslots e sem canal de controlo. Os terminais
são enviados para ela por atribuição de canal a partir da primária. Regras:

- `cell` tem de indicar um canal TETRA primário existente, e esse primário tem de ter `[channel.tetra.packet_data]`.
- `mcc`, `mnc`, `location_area` e `colour_code` têm de ser iguais aos do primário.
- Uma carrier de packet data não tem tabela `packet_data` própria, e há no máximo uma por primário.
- As suas frequências têm de estar na banda do primário, com o mesmo duplex spacing.
- Quando está desativada (`enabled = false`), a célula mantém o packet data apenas na carrier primária.

### 8.5 `[network.brew]`: ligação ao núcleo TETRA (Brew / TetraPack)

Esta tabela exige pelo menos um canal TETRA ativado.

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `host` | obrigatório | core.tetrapack.online | Nome de host ou IP do servidor. |
| `port` | 443 | 443 | Porta do servidor. |
| `tls` | true | true | Usar TLS (wss/https). |
| `username` / `password` | nenhum | definida | Autenticação HTTP Digest. Ou ambos, ou nenhum. |
| `reconnect_delay_secs` | 15 | 15 | Pausa antes de voltar a ligar (1–300). |
| `jitter_initial_latency_frames` | 0 | — | Delay inicial de reprodução adicional, em frames, para a voz de rede. |
| `feature_sds_enabled` | true | — | Passar mensagens SDS entre terminais locais e de rede. |
| `feature_rssi_export` | false | — | Comunicar o RSSI ao servidor. |
| `whitelisted_ssis` | nenhum | — | Permitir apenas chamadas de rede com estes SSIs remotos (até 4096 entradas, cada uma no máximo 0xFFFFFF). |

As chaves antigas `network.brew_endpoint` e `network.brew_token` ainda são interpretadas, mas o `doctor` e um arranque em funcionamento
rejeitam-nas. Use `[network.brew]` em alternativa.

## 9. DMR

### 9.1 `[channel.dmr]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `id` | obrigatório | 1234567 | ID de repeater de 24 bits em RF (1–16777215). É também o ID de rede, a menos que `network.id` esteja definido. |
| `colour_code` | obrigatório | 1 | Colour code (0–15). |

### 9.2 `[channel.dmr.worker]`: modem nativo (todas as chaves obrigatórias)

| Chave | Estação | Significado e regras |
|---|---|---|
| `symbol_deviation` | 10.0 | Escala FM dos símbolos 4FSK, usada no condicionamento de RX e no TX. Tem de ser > 0. |
| `tx_level_q15` | −13056 | Ganho de deviation de TX em Q15 com sinal. −13056 = −102 × 128, o nível SDR de referência. Um valor negativo inverte a deviation. |
| `tx_peak` | 0.5 | Envolvente de pico em banda base, 0 < x ≤ 0.8. |
| `power_calibration` | 0 | Offset somado à leitura de RSSI em dB. |
| `receiver_delay` | 3 | Delay por slot do recetor do repeater. A sua unidade exata não está documentada nesta base de código; mantenha 3. |
| `marker_offset_numerator` / `marker_offset_denominator` | 720 / 1 | Correção entre o marcador de temporização de TX e o RX condicionado, em amostras a 24 kS/s: 720 = 30 ms. O denominador não pode ser 0. |
| `maximum_input_age_ms` | 100 | Entrada de RX mais antiga que o worker ainda aceita. Tem de ser > 0. |
| `stall_ms` | 5000 | Timeout de bloqueio (stall) do worker. Tem de ser > 0. |
| `hang_frames` | 51 | Hang frames enviadas após um terminador de chamada (0–60). |

### 9.3 `[channel.dmr.network]`: ligação MMDVM homebrew

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `profile` | obrigatório | brandmeister | `brandmeister`, `dmrplus`, `tgif`, `freedmr` ou `custom` (um master próprio). Todos falam o protocolo MMDVM homebrew; só o perfil BrandMeister acrescenta as verificações de `version`/`software` abaixo. A página Settings oferece os masters da rede escolhida a partir da lista descarregada (ver `[dashboard.host_lists]`). |
| `endpoint` | obrigatório | 2262.master.brandmeister.network:62031 | `host:port` do master, resolvido quando o canal arranca. |
| `password` | obrigatório | definida | Palavra-passe de autenticação (1–256 bytes). |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62002 | Endereço UDP local. A porta 0 deixa o sistema escolher. |
| `callsign` | obrigatório | N0CALL | 1–8 caracteres. |
| `id` | `[channel.dmr] id` | 123456701 | ID de rede. O ID efetivo tem de ser > 1000. |
| `name` | derivado do host | — | Nome apresentado no dashboard. |
| `retry_ms` | 5000 | 1000 | Intervalo entre tentativas de autenticação. Tem de ser > 0. |
| `timeout_ms` | 15000 | 60000 | Timeout da sessão. Tem de ser maior do que `retry_ms`. |

### 9.4 `[channel.dmr.network.runtime]`: dados da estação e política de sessão

Todas as chaves são obrigatórias, exceto `rf_inactivity_timeout_ms`.

| Chave | Estação | Significado e regras |
|---|---|---|
| `power`, `latitude`, `longitude`, `height` | 0, 0.0, 0.0, 0 | Dados da estação enviados na autenticação: potência 0–99, latitude ±90, longitude ±180, altura 0–999. |
| `location` | "" | Até 20 caracteres (texto mais longo é cortado). |
| `description` | "Nexus-BS 2.0 hotspot" | Até 19 caracteres (texto mais longo é cortado). |
| `url` | "" | Até 124 caracteres. |
| `version` | 20260916_Nexus | Com BrandMeister tem de ser `YYYYMMDD_…`. |
| `software` | MMDVM_Nexus | Com BrandMeister tem de começar por `MMDVM`; caso contrário, o master recusa a autenticação. |
| `options` | "" | String de opções enviada ao master, p. ex. `TS1=1;TS2=1`. |
| `retention_ms` | 250 | Retenção da voz de rede. Tem de ser > 0. |
| `rf_inactivity_timeout_ms` | 5000 | Timeout de inatividade de RF. Omita-o, ou defina um valor > 0. |
| `embedded_lc_only` | true | true = enviar apenas o embedded link control na voz de rede; false = manter os dados embedded recebidos. |

## 10. P25

### 10.1 `[channel.p25]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `nac` | obrigatório | 0x293 | Network Access Code, 0x000–0xFFF. Escreva-o em hexadecimal, tal como está programado nos rádios. O dashboard mostra-o em decimal (659). |

### 10.2 `[channel.p25.worker]`: modem nativo (todas as chaves obrigatórias)

| Chave | Estação | Significado e regras |
|---|---|---|
| `symbol_deviation`, `tx_level_q15`, `tx_peak` | 10.0, −13056, 0.5 | Como no DMR (§9.2). |
| `duplex` | true | Transmissor duplex. `tx_hang` só funciona em duplex. |
| `tx_delay` | 0 | Unidades MMDVM originais: 500 ms + valor × 10 ms, limitado a 1 s. |
| `tx_hang` | 0 | Segundos de hang após uma chamada. |
| `network_status` | inbound_outbound | Bits de estado enviados no ar: `inbound_busy`, `inbound_idle` ou `inbound_outbound`. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Como no DMR. |
| `assembly_ms` | 500 | Janela para montar as frames de voz do reflector (LDUs). |
| `retention_ms` | 1000 | Durante quanto tempo uma LDU completa é retida para TX. Tem de ser menor do que `reflector.timeout_ms`. |
| `watchdog_ms` | 2000 | Uma chamada termina após este tempo sem tráfego. |
| `call_limit_ms` | 180000 | Limite absoluto de duração de uma chamada. |

### 10.3 `[channel.p25.reflector]`

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `endpoint` | obrigatório | p25.tetralink.ro:41000 | `host:port` do reflector. |
| `bind` | `0.0.0.0:0` | 0.0.0.0:62005 | Endereço UDP local. |
| `callsign` | obrigatório | N0CALL | 1–10 caracteres, apenas `A-Z 0-9 / -` em maiúsculas. |
| `interval_ms` | 5000 | 5000 | Intervalo de polling. |
| `timeout_ms` | 15000 | 15000 | Timeout da ligação. Tem de ser maior do que `interval_ms`. |
| `talkgroup` | nenhum | — | Rota fixa opcional: apenas as group calls de RF para este talkgroup vão para o reflector, e o tráfego recebido é reescrito para ele. Ausente significa transparente. 0 é inválido. |
| `name` | derivado do host | — | Nome apresentado. |
| `password` | — | — | Não suportada pelo protocolo do reflector P25: defini-la é um erro. |
| `auto_select` | false | — | Seleciona o reflector a partir do talkgroup marcado em RF, como faz o P25Gateway. Uma group call para outro talkgroup encontrado em `hosts` ou na lista de reflectores descarregada liga esse reflector entre chamadas: o over que o seleciona fica local, o seguinte sai para a rede. Requer `talkgroup` (o valor predefinido para o qual regressa). |
| `revert_secs` | 600 | — | Segundos sem tráfego antes de regressar ao `endpoint`/`talkgroup` configurado. 0 mantém-se no último reflector selecionado. |
| `[[…reflector.hosts]]` | nenhum | — | Entradas de reflector próprias, `talkgroup` (1–65535), `endpoint` (`host:port`) e `name` opcional; têm prioridade sobre a lista descarregada. Até 256. |

Exemplo com seleção automática:

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

## 11. FM analógico

O canal FM é o repeater FM MMDVM original a funcionar no SDR, ligado em ponte a uma rede através do svxlink
(RoLink) ou de uma ligação USRP legada a 8 kHz. O canal de RF é de 25 kHz.

### 11.1 `[channel.fm]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `callsign` | obrigatório | N0CALL | Callsign da estação (1–23 caracteres imprimíveis), usado nos metadados de rede. O texto do CW ID é `keyers[0].text`. |

### 11.2 `[channel.fm.modem]`: repeater FM MMDVM (todas as chaves obrigatórias)

| Chave | Estação | Significado e regras |
|---|---|---|
| `mode` | simplex | `link`, `simplex` ou `duplex`. |
| `access` | carrier_and_ctcss | Como a entrada abre: `carrier`, `ctcss_delayed`, `carrier_and_ctcss` ou `ctcss_latch`. |
| `external_enabled` | true | Ativar o caminho de áudio de rede. |
| `rx_level` | 200 | Escala da entrada de RF, 256 / `rx_level`. Não pode ser 0. Com 200 há margem; com 128 as estações fortes saturavam (clipping). |
| `tx_level` | 128 | Escala de saída. As unidades de `tx_ceiling` do svxlink pressupõem 128. |
| `rf_boost` / `external_boost` | 1 / 1 | Multiplicadores de ganho ao retransmitir áudio de RF ou de rede. |
| `ctcss_frequency` | 103 | Tom CTCSS como Hz inteiros da tabela MMDVM (103 = 103.5 Hz). Códigos válidos: 67 69 71 74 77 79 82 85 88 91 94 97 100 103 107 110 114 118 123 127 131 136 141 146 151 156 159 162 165 167 171 173 177 179 183 186 189 192 196 199 203 206 210 218 225 229 233 241 250 254. |
| `ctcss_high` / `ctcss_low` | 150 / 100 | Limiares do detetor CTCSS, em unidades MMDVM originais. |
| `ctcss_level` | 16 | Nível do tom CTCSS em TX: cerca de 400 Hz de deviation nesta estação. |
| `squelch_high` / `squelch_low` | 111 / 105 | Limiares do carrier squelch sobre a métrica de RSSI. O squelch abre ou fecha após 4 leituras consecutivas para lá de um limiar. Com `cos_invert = true`, abre em `squelch_low` ou abaixo e fecha em `squelch_high` ou acima. |
| `cos_invert` | true | Inverte a comparação. Nesta placa a métrica mede ruído, pelo que um sinal forte dá um valor *baixo*. Idle floor medido: 115.3. |
| `max_deviation` | 0 | Limiar de blanking por over-deviation × 128; 0 desliga-o. Substitui o áudio excessivo por um bip e silêncio; **não** é um limiter. Não prova nada sobre a deviation de RF. |
| `timeout_level` | 20 | Nível do tom de timeout e do bip de blanking. |
| `callsign_at_start` / `callsign_at_end` / `callsign_at_latch` | false | Quando enviar o CW ID. |

**`[channel.fm.modem.timers]`**: todas as chaves obrigatórias, em milissegundos; 0 desativa um temporizador.

| Chave | Estação | Significado |
|---|---|---|
| `callsign` | 600000 | Intervalo do CW ID (10 min). |
| `timeout` | 180000 | Timeout de TX (3 min). |
| `holdoff` | 0 | Temporizador de hold-off. |
| `kerchunk` | 250 | Temporizador de kerchunk. |
| `ack_min` / `ack_delay` | 1000 / 500 | Temporizadores de acknowledgement. |
| `hang` | 300 | Hang time do repeater. |

A semântica MMDVM exata de `holdoff`, `kerchunk` e dos temporizadores de acknowledgement é a do controlador FM MMDVM
original. Esta base de código não a documenta mais.

**`[[channel.fm.modem.keyers]]`**: exatamente três blocos, por esta ordem: callsign, acknowledgement de RF, acknowledgement
de rede.

| Chave | Estação | Significado |
|---|---|---|
| `text` | N0CALL / K / R | Texto CW, até 255 caracteres; os caracteres desconhecidos são ignorados. |
| `speed_wpm` | 20 / 18 / 22 | Velocidade; não pode ser 0. |
| `frequency_hz` | 1000 / 900 / 1100 | Frequência do tom, 1–24000. |
| `high_level` / `low_level` | 0 / 0 | Níveis do tom do keyer em unidades MMDVM originais. A base de código não os documenta com precisão. |

### 11.3 `[channel.fm.worker]`: caminho do sinal (todas as chaves obrigatórias, exceto `squelch`)

| Chave | Estação | Significado |
|---|---|---|
| `symbol_deviation` | 10.0 | Escala da modulação FM. Tem de ser > 0. |
| `rx_dc_block` | true | Bloqueador de DC antes da desmodulação. |
| `power_calibration` | 0 | Offset somado à leitura de RSSI em dB. |
| `tx_peak` | 0.5 | Envolvente de pico em banda base, 0 < x ≤ 0.8. |
| `host_tx_gain` | 1.0 | Ganho de áudio RF → rede. Tem de ser > 0. |
| `host_rx_gain` | 9.0 | Ganho de áudio rede → RF. Tem de ser > 0. Nesta estação o svxlink envia áudio plano, sem limitação, e o nexus2 faz o processamento. |
| `pre_emphasis` / `de_emphasis` | false / false | Filtros do lado do host. Desligados aqui porque as opções svxlink abaixo fazem esse trabalho. |
| `maximum_input_age_ms`, `stall_ms` | 100, 5000 | Como no DMR. |

**`[channel.fm.worker.squelch]`** (opcional; todas as chaves têm valor predefinido):

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `mode` | fixed | fixed | `fixed`: decidem os limiares do modem acima. `shadow`: apenas medir e reportar. `adaptive`: decide um tracker do noise floor. `adaptive` exige `squelch_high = 1`, `squelch_low = 0` e `cos_invert = false`; isto é verificado no arranque, não pelo `doctor`. |
| `open_db` / `close_db` | 10 / 5 | — | SNR (signal-to-noise ratio) necessário para abrir e para se manter aberto (0–60; close < open). Usado por `adaptive`. |
| `quiet_open_db` / `quiet_close_db` | 6 / 3 | — | Quieting necessário para abrir e para se manter aberto (0–60; close < open). |

### 11.4 `[channel.fm.svxlink]`: ponte para svxlink / RoLink

Esta ponte envia PCM bruto a 16 kHz sobre UDP, com squelch e PTT através de PTYs. É exclusiva com `[channel.fm.usrp]`:
configure exatamente uma.

| Chave | Predefinido | Estação | Significado e regras |
|---|---|---|---|
| `rx_audio` | obrigatório | 127.0.0.1:40100 | Para onde é enviado o áudio de RF. Corresponde ao svxlink `[Rx] AUDIO_DEV=udp:…`. |
| `tx_audio_bind` | obrigatório | 127.0.0.1:40101 | Socket local para o áudio de rede vindo do svxlink `[Tx] AUDIO_DEV=udp:…`. Tem de ser diferente de `rx_audio`. |
| `ptt_pty` | obrigatório | …/var/run/ptt | svxlink `[Tx] PTT_PTY`. |
| `squelch_pty` | obrigatório | …/var/run/sql | svxlink `[Rx] PTY_PATH` (`SQL_DET=PTY`). Tem de ser diferente de `ptt_pty`. |
| `talker_file` | nenhum | …/var/run/talker | Ficheiro onde o svxlink escreve o talker atual, apresentado no dashboard. |
| `tg_file` | nenhum | …/var/run/tg | Ficheiro com o talkgroup selecionado. |
| `conf` | nenhum | …/svxlink.conf | Configuração própria do svxlink, lida uma vez no arranque para o nome da rede, o callsign e os talkgroups apresentados. |
| `network` | derivado do host do reflector | RoLink | Nome apresentado. |
| `jitter_ms` | 75 | 75 | Prebuffer do áudio recebido (0–500). |

**Processamento de RX (RF → rede).** Todas as etapas estão desligadas por predefinição; a estação usa o conjunto recomendado.

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `rx_de_emphasis_hz` | 0 (desligado) | 300.0 | Frequência de canto da de-emphasis, 0 ou 50–2000 Hz. |
| `rx_gate_depth_db` | 0 (desligado) | 30.0 | Profundidade do downward expander que segue o noise floor (0–60). As chaves abaixo só se aplicam quando este valor é > 0. |
| `rx_gate_open_db` / `rx_gate_close_db` | 10 / 5 | — | Nível acima do floor que conta como fala (3–40) e que a mantém (1…open). |
| `rx_gate_hold_ms` / `rx_gate_attack_ms` / `rx_gate_release_ms` | 120 / 5 / 60 | — | Temporização do gate (hold ≤ 2000, attack 1–200, release 5–2000). |
| `rx_floor_rise_db_per_s` | 5.0 | — | Velocidade máxima de subida da estimativa do floor (0.1–60). |
| `rx_level_target_dbfs` | desligado | −21.0 | Alvo do leveler, rms da fala (−40…−6, abaixo do teto de pico). As chaves seguintes só se aplicam quando este está definido. |
| `rx_level_initial_gain_db` / `rx_level_min_gain_db` / `rx_level_max_gain_db` | 20 / −10 / 30 | — | Ganhos do leveler (min −20…max, max 0–40). |
| `rx_level_rise_db_per_s` / `rx_level_fall_db_per_s` | 8 / 30 | — | Velocidade do leveler (0.5–60 / 1–120). |
| `rx_peak_ceiling_dbfs` | desligado | −6.0 | Teto do limiter look-ahead (−20…−0.5). |

**Processamento de TX (rede → RF):**

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `tx_pre_emphasis` | false | true | O nexus2 aplica pre-emphasis (+6 dB/oitava acima de 300 Hz, unitária a 1 kHz). Defina em conjunto o svxlink `[Tx1] PREEMPHASIS=0`. |
| `tx_limiter` | false | true | Peak limiter look-ahead em `tx_ceiling`. Defina em conjunto o svxlink `LIMITER_THRESH=0`. |
| `tx_ceiling` | 2048 | 2200 | Teto do limiter (256–3072). 1 unidade ≈ 1.9 Hz de deviation com `tx_level` 128, pelo que 2200 ≈ 4.2 kHz. |

### 11.5 `[channel.fm.usrp]`: ligação USRP legada (alternativa ao svxlink)

Todas as chaves são obrigatórias. A ligação transporta áudio legado a 8 kHz.

| Chave | Significado |
|---|---|
| `bind` | `ip:port` UDP local. Não pode partilhar uma porta diferente de zero com um canal FM anterior. |
| `peer` | `ip:port` unicast remoto, da mesma família de endereços que `bind` e diferente dele. |
| `loss_timeout_ms` | Após este tempo sem pacotes, a voz de rede é libertada. Tem de ser > 0. |

### 11.6 `[callsigns]`: identidades e MDC1200

Esta tabela é usada apenas pela ponte FM svxlink. É carregada no arranque e reconstruída com `nexus2 callsigns`.

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `enabled` | true | true | Usar de todo as identidades. |
| `database` | nenhum | /opt/nexus-bs2/callsigns.csv | Ficheiro `callsign,mdc_id`. Tem de ser legível no arranque; caso contrário, o arranque falha. |
| `source` | nenhum | API de nós RoLink | Lista de operadores usada na reconstrução: um URL ou um ficheiro local. |
| `selection` | active | active | `active` = operadores ouvidos nos últimos `active_days`; `all` = todos os operadores alguma vez listados. |
| `active_days` | 30 | 60 | Janela retrospetiva (1–3650). |
| `mdc_encode` | false | true | Enviar o ID MDC1200 do talker da rede antes do áudio de rede em FM analógico. Apenas no sentido de saída: o MDC ouvido em RF nunca é considerado fiável. |
| `unknown` | RoLink | ROLINK | Etiqueta anunciada para um talker que não consta da base de dados. Vazio não envia nada. |

## 12. Modos adiados

Os canais `dstar`, `ysf`, `nxdn` e `pocsag` são interpretados e aparecem em `plan`/`doctor`, mas um arranque em funcionamento recusa-os
enquanto estiverem ativados. As suas tabelas aceitam `tx_delay` (os quatro), `low_deviation` e `hang_seconds` (YSF, predefinido 4) e
`hang_seconds` (NXDN, predefinido 5). Existem apenas para planeamento.

## 13. `[dashboard]`, `[network]`, `[touch]`

### `[dashboard]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `listen` | 127.0.0.1:8872 | 0.0.0.0:8080 | Endereço HTTP e WebSocket. Um endereço que não seja de loopback exige `auth.enabled = true`. |
| `id_database_url` | https://radioid.net/static/user.csv | — | Origem da base de dados de callsign/nome/país (apenas https). Atualizada quando tem mais de 7 dias. |
| `id_database_dir` | `<config dir>/ids` | — | Onde essa base de dados é guardada. |
| `host_lists.dmr_url` | https://www.pistar.uk/downloads/DMR_Hosts.txt | — | Lista de masters DMR (formato Pi-Star) para a página Settings. |
| `host_lists.p25_url` | https://refcheck.radio/api/hostfile-gate/fetch/p25/ | — | Lista de reflectores P25 (formato P25Hosts.txt): o registo DVRef, servido pela RefCheck.Radio. Também usada pelo `auto_select` do P25. |
| `host_lists.p25_token` | nenhum | — | O seu token pessoal RefCheck.Radio: introduza o seu callsign em https://hostfiles.refcheck.radio para obter um. Sem ele, a lista P25 não é descarregada (as suas entradas `hosts` definidas manualmente continuam a funcionar). Os termos deles permitem uma descarga por hora; a lista credita a DVRef. Removido das exportações Share. |
| `host_lists.refresh_hours` | 24 | — | Volta a descarregar as listas após este número de horas (1–720). As cópias em cache ficam junto à base de dados de ID. |
| `auth.enabled` | true | true | Autenticação HTTP Basic em todas as páginas, rotas da API e WebSocket. |
| `auth.username` | admin | definida | 1–64 caracteres imprimíveis, sem `:`. |
| `auth.password` | nexus | definida | 1–256 caracteres. **Altere o valor predefinido.** |

### `[network]`

| Chave | Predefinido | Estação | Significado |
|---|---|---|---|
| `connect_timeout_secs` | 10 | 10 | Validada (1–300), mas atualmente não usada por nenhuma ligação. |
| `brew` | nenhum | definida | Ligação ao núcleo TETRA, §8.5. |

### `[touch]`: painel Nexus-BS Touch

Esta tabela é lida apenas pelo painel, que a volta a ler até 5 s após uma alteração.

| Chave | Predefinido | Significado |
|---|---|---|
| `backlight` | 100 | % de retroiluminação em uso (1–100). |
| `screen_timeout_secs` | 300 | Segundos de inatividade antes da proteção de ecrã; 0 = sempre ligado (máximo 86400). |
| `screensaver` | splash | `splash` = o logótipo de arranque, estático e esbatido; `blank` = preto com a retroiluminação desligada. |
| `screensaver_dim` | 30 | % de brilho do logótipo esbatido. |
| `screensaver_backlight` | 25 | % de retroiluminação enquanto o logótipo é mostrado. |
| `wake_on_touch` | true | Um toque acorda o ecrã. Esse toque é consumido e nunca aciona um controlo. |
| `wake_on_traffic` | true | Uma nova chamada ou entrada de last-heard acorda o ecrã. |

Com um timeout definido, pelo menos uma das duas opções de despertar tem de estar ligada. O controlo da retroiluminação exige
`/sys/class/backlight` e permissão de escrita; sem eles, o painel apenas escurece a imagem.

## 14. Linha de comandos

`--config <file>` tem como predefinido `nexus2.toml`.

| Comando | Finalidade |
|---|---|
| `nexus2 plan` | Mostrar o plano de frequências sem hardware. |
| `nexus2 doctor [--live]` | Validar tudo o que pode ser verificado offline. `--live` acrescenta os requisitos do arranque em funcionamento (§6). |
| `nexus2 plan-risks --occupied-half-bandwidth-hz N --rx-bandwidth-hz N --tx-bandwidth-hz N [--sample-rate N]` | Geometria offline de DC, imagem e intermodulação do plano. |
| `nexus2 init [--defaults]` | Criar um ficheiro novo (TX inibido). Recusa sobrescrever um já existente. |
| `nexus2 callsigns --out FILE [...]` | Reconstruir a base de dados de callsigns/MDC1200 a partir de `[callsigns]`. |
| `nexus2 run --live [--allow-rf-tx]` | A própria estação (o serviço systemd). |
| `nexus2 manage ...` | A API de gestão HTTPS (serviço separado). |

## 15. Receitas

**Mover uma carrier.** Altere o seu `offset_hz` num múltiplo de 25 kHz. Mantenha 50 kHz em relação às vizinhas e fique dentro de
±(sample rate / 2 − 20 kHz). Execute `nexus2 plan`, depois `doctor --live` (o TETRA tem de manter uma codificação normalizada), aplique e
reinicie.

**Tirar um modo do ar em definitivo.** Defina `enabled = false` no respetivo `[[channel]]`. Mantém a sua posição de frequência. Para
DMR, P25 ou FM, uma desativação temporária pode ser feita em funcionamento a partir do dashboard, e dura até ao próximo reinício.

**Ativar a carrier de packet data TETRA.** Defina `enabled = true` em `tetra-data`, mantendo a sua identidade igual à do
primário, e reinicie. O primário mantém o seu `[channel.tetra.packet_data]`.

**Ajustar o nível de TX de um modo.** Altere o `tx_peak` desse modo: `[channel.tetra.worker]`, `[channel.dmr.worker]`,
`[channel.p25.worker]` ou `[channel.fm.worker]`. Mantenha-o em 0.8 ou abaixo. O total de todas as carriers partilha um único DAC, pelo que
aumentar um modo aumenta o pico composto. `sdr.tx_gain_db` e os ganhos por elemento movem **todos** os modos em conjunto.

**Medir os níveis de RX.** Defina `sdr.io_diagnostics = true`, reinicie, recolha as linhas `RX_LEVEL` do journal e depois volte a
defini-lo como `false`.

**Alterar a palavra-passe do dashboard.** Defina `dashboard.auth.password` e reinicie o serviço de rádio. O painel tátil
autentica-se no dashboard com as credenciais deste mesmo ficheiro, mas só as lê uma vez, quando arranca, por isso
reinicie também `nexus-panel.service`.

## 16. Limitações conhecidas e pontos em aberto

Resultam do código tal como está; são indicadas para que ninguém conte com elas:

- `network.connect_timeout_secs` é validada, mas nenhuma ligação a usa.
- O limite de 0.8 para o `tx_peak` TETRA documentado no código não é imposto; apenas 0–1.0 é, quando o TX está ativado.
- `packet_data.max_slots` fora de 1–4 é limitado silenciosamente.
- `sdr.driver` não seleciona o dispositivo (quem o faz é `uri`). Um nome de driver errado apenas altera o comportamento
  específico do SX1255.
- `nexus2 callsigns` reverte discretamente para os valores predefinidos se o ficheiro não for carregado, e pode sondar o SDR quando `[sdr]`
  usa auto.
- Não documentado nesta base de código: as unidades exatas do `receiver_delay` do DMR; o que exatamente aciona `stall_ms`; as escalas
  de `high_level`/`low_level` do keyer FM e dos limiares/nível de CTCSS (unidades de byte MMDVM originais); o texto CW mais longo que
  cabe.
- O recarregamento de definições por canal sem reinício existe no código, mas não tem nenhum chamador em produção. Apenas a comutação
  ligar/desligar de DMR, P25 e FM funciona em funcionamento.
