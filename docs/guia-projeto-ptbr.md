# Guia completo (PT-BR): arquitetura, instalação, gravação na placa, Bluetooth e MQTT

Este documento é um **guia prático** para quem quer começar no `xiaozhi-esp32` com foco em:

- como compilar e subir firmware para a placa;
- como escolher o tamanho correto de partição (flash 4MB/8MB/16MB/32MB);
- como o projeto está organizado;
- como testar o básico no dia a dia;
- como usar Bluetooth (BluFi) e MQTT.

> Dica rápida: se você ainda não definiu a sua placa exata, comece em `idf.py menuconfig` -> `Board Type`.

---

## 1) Pré-requisitos

1. **ESP-IDF 5.4+** (conforme README do projeto).
2. Python e toolchain instalados via ESP-IDF.
3. Cabo USB de dados e porta serial funcional.
4. Linux recomendado para build mais estável/rápido.

Referência no projeto:

- `README.md` (seção *Development Environment*)

---

## 2) Como subir o projeto para sua placa (passo a passo)

### 2.1 Defina o alvo do chip

Escolha conforme seu hardware:

```bash
idf.py set-target esp32s3
# ou
idf.py set-target esp32c3
# ou
idf.py set-target esp32
```

### 2.2 Abra configuração do projeto

```bash
idf.py menuconfig
```

No menu, configure principalmente:

1. **Board Type**: selecione o modelo da sua placa.
2. **Método de provisionamento Wi‑Fi** (SmartConfig/AP/BluFi).
3. **Transporte de comunicação** (WebSocket ou MQTT+UDP, dependendo da sua stack).

### 2.3 Escolha o tamanho correto de partição da flash

Se sua placa tem flash diferente, ajuste a tabela de partições:

- `partitions/v2/4m.csv`  -> placas de **4MB**
- `partitions/v2/8m.csv`  -> placas de **8MB**
- `partitions/v2/16m.csv` -> placas de **16MB**
- `partitions/v2/32m.csv` -> placas de **32MB**

Em builds customizados, isso aparece como macro:

`CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions/v2/<tam>.csv"`

> Se você não souber o tamanho da flash, confirme no datasheet da placa/módulo antes de gravar.

### 2.4 Build + flash + monitor

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

No Windows, troque `/dev/ttyUSB0` por `COMx`.

### 2.5 Se algo der errado

Limpe e reconstrua:

```bash
idf.py fullclean
idf.py build
```

---

## 3) Arquitetura do projeto (visão prática)

## 3.1 Pastas principais

- `main/`
  - código principal do firmware.
- `main/boards/`
  - implementação por placa (pinagem, display, áudio, botões, etc.).
  - cada board costuma ter um `config.json` com opções de build.
- `main/audio/`
  - pipeline de áudio, codecs e wake word.
- `main/display/`
  - abstrações de display (OLED/LCD/LVGL/emotes).
- `main/protocols/`
  - protocolos de transporte e integração com backend.
- `main/assets/`
  - strings, fontes, recursos embarcados.
- `docs/`
  - documentação técnica (board, MQTT, websocket, BluFi, MCP).
- `partitions/v2/`
  - tabelas de partição por tamanho de flash.
- `scripts/`
  - scripts de build/empacotamento/utilidades.

## 3.2 Fluxo de inicialização (resumo)

1. Seleção da board por `menuconfig` (`CONFIG_BOARD_TYPE_*`).
2. `main/CMakeLists.txt` resolve `BOARD_TYPE` e inclui arquivos específicos.
3. Inicialização de periféricos da placa (áudio/display/rede).
4. Inicialização do protocolo de comunicação (WebSocket ou MQTT+UDP).
5. Loop de operação (captura de áudio, envio, respostas, UI/estado).

## 3.3 Onde mexer quando quiser customizar

- Nova placa: `main/boards/<sua-board>/`
- Layout/face/UI: `main/display/` e implementação da board
- Transporte/rede: `main/protocols/`
- Strings e recursos: `main/assets/`

---

## 4) Dicas e macetes que economizam tempo

1. **Não altere board existente para hardware diferente**: crie uma nova board (evita quebrar builds alheios).
2. **Partição correta é crítica**: tabela errada = flash instável, OTA falhando ou boot quebrado.
3. **Padronize builds por board**: mantenha `config.json` por placa para reproduzir binários.
4. **Valide serial cedo**: se `flash` falhar, teste apenas conexão/baud/reset antes de depurar código.
5. **Use logs no monitor** para validar:
   - provisionamento Wi‑Fi;
   - conexão com backend;
   - transições de estado (listening/speaking/idle).

---

## 5) Como testar no dia a dia

> O projeto não centraliza testes unitários tradicionais; na prática embarcada, os testes mais úteis aqui são de build e smoke test em hardware.

### 5.1 Teste mínimo de CI local

```bash
idf.py build
```

### 5.2 Smoke test em placa

1. `idf.py flash monitor`
2. Verifique no log:
   - boot sem reset loop;
   - inicialização de áudio/display;
   - conexão de rede;
   - handshake do protocolo (WebSocket ou MQTT).

### 5.3 Checklist rápido antes de publicar firmware

- Board correta selecionada em menuconfig.
- Partição compatível com tamanho da flash.
- Conectividade estável no backend escolhido.
- Wake word / botão / áudio funcionando na placa alvo.

---

## 6) Como usar Bluetooth (BluFi)

Para provisionar Wi‑Fi por BLE (BluFi):

1. Em `idf.py menuconfig`, habilite **Esp Blufi** (`CONFIG_USE_ESP_BLUFI_WIFI_PROVISIONING=y`).
2. Garanta que só **uma** stack BT está ativa:
   - `CONFIG_BT_BLUEDROID_ENABLED` **ou** `CONFIG_BT_NIMBLE_ENABLED` (não as duas).
3. Compile e grave o firmware.
4. Use app compatível com BluFi para enviar SSID/senha via BLE.

Observações importantes:

- BluFi não deve ficar ativo ao mesmo tempo que outro método conflitante de provisionamento.
- Consulte `docs/blufi.md` para detalhes de configuração e limitações.

---

## 7) Como usar MQTT

O projeto suporta arquitetura híbrida **MQTT + UDP**:

- **MQTT**: controle, estado e mensagens JSON.
- **UDP**: tráfego de áudio em tempo real.

Passos práticos:

1. Selecione transporte MQTT (conforme configuração do firmware/backend).
2. Garanta endpoint e credenciais válidas no backend.
3. Faça flash e abra monitor serial.
4. Verifique no log:
   - conexão MQTT estabelecida;
   - `hello`/resposta com parâmetros UDP;
   - envio/recebimento de mensagens de sessão.

Para entendimento do protocolo e sequência de mensagens, use:

- `docs/mqtt-udp.md`
- `docs/mcp-protocol.md`

---

## 8) Referências diretas dentro do projeto

- Guia de board customizada: `docs/custom-board.md`
- BluFi: `docs/blufi.md`
- MQTT+UDP: `docs/mqtt-udp.md`
- WebSocket: `docs/websocket.md`
- MCP: `docs/mcp-usage.md` e `docs/mcp-protocol.md`
- Partições v2: `partitions/v2/README.md`

---

Se você quiser, no próximo passo eu posso criar uma versão **específica da sua placa** (com chip, tamanho de flash, porta serial e comandos exatos prontos para copiar/colar).
