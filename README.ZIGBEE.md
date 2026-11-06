# Panasonic AC via Zigbee (ESP32-C6 Super Mini)

Configuração de exemplo: [`ac-zigbee-esp32c6.yaml.example`](ac-zigbee-esp32c6.yaml.example)

O ESP32-C6 controla o AC pela UART e ao mesmo tempo funciona como roteador (repetidor) Zigbee. O Wi-Fi fica desligado e só é ligado, por um switch Zigbee, para atualizações OTA.

## Requisitos

* ESP32-C6 Super Mini (ou outra placa ESP32-C6/H2, ajustando os pinos)
* **ESPHome 2026.10 ou mais novo**, com o Zigbee nativo. Até a 2026.9 o Zigbee do ESP32 só exporta `sensor` e `binary_sensor`, então os switches e numbers (os controles do AC) não apareceriam. Não use o componente externo `luar123/zigbee_esphome`.
* Coordenador Zigbee: ZHA ou Zigbee2MQTT

> O nRF52840 não serve para o AC hoje: o ESPHome ainda não tem driver de UART para o nRF52. Ele pode ser usado como roteador Zigbee.

## Ligação

| Fio do AC | Super Mini |
| --- | --- |
| TX do AC (5 V) | divisor de tensão → pino RX (GPIO17) |
| RX do AC | pino TX (GPIO16), direto (o AC aceita 3,3 V) |
| 5 V | 5V |
| GND | GND |

Atenção: nas fotos e tabelas de [CZ-TACG1](README.CZTACG1.md) e [DNSK-P11](README.DNSKP11.md), RX/TX estão nomeados do ponto de vista do ESP. O fio marcado **RX** nelas é o que sai do AC (o TX do AC) e é ele que leva o divisor.

Divisor de tensão: resistor de 1 kΩ em série entre o TX do AC e o GPIO17, e 2 kΩ entre o GPIO17 e o GND (5 V × 2/3 ≈ 3,3 V). Um optoacoplador PC817 também funciona a 9600 baud.

**O ESP32-C6 não aceita 5 V nos pinos.** Com o AC ligado e nada conectado, meça o pino RX do AC: se der cerca de 5 V, existe um pull-up interno e convém colocar um resistor de ~1 kΩ em série no TX do ESP.

O 5 V do conector do AC vai no pino 5V da Super Mini (nunca no 3V3). Não ligue a USB do computador ao mesmo tempo que o 5 V do AC.

Evite os pinos GPIO4 e 5 (boot), GPIO8 (LED RGB e boot), GPIO9 (botão BOOT), GPIO15 (LED azul e boot) e GPIO12 e 13 (USB).

## Entidades no Zigbee

O Zigbee do ESPHome não exporta a entidade `climate`. Ela existe só dentro do firmware, e estas entidades refletem e controlam o AC:

| Entidade | Tipo | Valores |
| --- | --- | --- |
| `ac_power` | switch | liga no último modo usado / desliga |
| `ac_target_temperature` | number | 16 a 30 °C, passo 0,5 |
| `ac_mode` | number | 0 desligado, 1 auto, 2 frio, 3 quente, 4 seco, 5 ventilar |
| `ac_fan` | number | 0 auto, 1 a 5 velocidade |
| `ac_preset` | number | 0 normal, 1 powerful, 2 quiet |
| `ac_vswing` | number | 0 swing, 1 auto, 2 up, 3 up_center, 4 center, 5 down_center, 6 down |
| `ac_hswing` | number | 0 auto, 1 left, 2 left_center, 3 center, 4 right_center, 5 right |
| `ac_current_temperature` | sensor | temperatura ambiente lida pelo AC |
| `ac_outside_temperature` | sensor | temperatura externa |
| `ac_defrost` | binary_sensor | degelo ativo |
| `wifi_ota` | switch | liga o Wi-Fi para OTA |
| `heartbeat`, `uptime_minutes`, `chip_temperature` | sensor | diagnóstico |

Os números só mudam de valor depois que o AC confirma o comando.

No Home Assistant dá para juntar tudo de novo em um termostato com o "Template Climate" (HACS) ou com um conversor externo no Zigbee2MQTT.

## Instalação passo a passo

### 1. Material

* ESP32-C6 Super Mini
* Resistores de 1 kΩ e 2 kΩ (ou 2 de 1 kΩ em série no lugar do 2 kΩ)
* Fios, termorretrátil e ferro de solda
* Conector ou fios para o CN-CNT / CN-WLAN do AC (veja os guias de hardware acima)
* Cabo USB-C com dados

### 2. Preparar o ESPHome

1. No Home Assistant, atualize o add-on **ESPHome Device Builder** para 2026.10 ou mais novo e abra-o (ou use o ESPHome pela linha de comando).
2. Crie um dispositivo novo e cole o conteúdo de [`ac-zigbee-esp32c6.yaml.example`](ac-zigbee-esp32c6.yaml.example).
3. Em **Secrets** (ou no arquivo `secrets.yaml`) adicione:
   ```yaml
   wifi_ssid: "NomeDaSuaRede"
   wifi_password: "SenhaDaSuaRede"
   api_encryption_key: "chave gerada pelo ESPHome"
   ```
   Se não quiser a API, remova o bloco `api:` do YAML.
4. Ajuste no YAML:
   * `type: cnt` para CN-CNT (CZ-TACG1) ou `type: wlan` para CN-WLAN (DNSK-P11)
   * descomente os switches extras (eco, nanoe X etc.) que o seu controle remoto tiver
5. Clique em **Validate**. Deve terminar sem erros (o aviso sobre roteador com Wi-Fi é esperado).

### 3. Primeira gravação (USB)

A primeira gravação precisa ser pela USB. Se a placa já tinha outro firmware Zigbee (por exemplo o `luar123/zigbee_esphome`), apague a memória para não sobrar dados da rede antiga.

1. Clique em **Install → Manual download → Factory format** e baixe o `.bin`.
2. Abra [web.esphome.io](https://web.esphome.io) no Chrome ou Edge e conecte a Super Mini na USB.
3. Clique em **Connect** e escolha a porta. Se a placa não aparecer, segure o botão **BOOT**, aperte e solte o **RST** e solte o BOOT.
4. Clique em **Install**, marque **Erase device** e escolha o `.bin`.
5. Ao terminar, aperte **RST** e abra **Logs**. Você deve ver `Panasonic AC component v... starting` e mensagens do Zigbee.

Pela linha de comando, o equivalente é `esphome run ac-zigbee-esp32c6.yaml` e escolher a porta USB.

### 4. Parear no Zigbee

1. Se a placa estava pareada como roteador antigo, remova esse dispositivo do Zigbee2MQTT / ZHA primeiro.
2. Ative o pareamento: **Permit join** no Zigbee2MQTT ou **Adicionar dispositivo** no ZHA.
3. Reinicie a Super Mini (RST ou tirando e pondo a USB). Ela entra na rede sozinha.
4. Confira se aparecem as entidades da tabela acima. Sem o AC ligado, os valores do AC ficam vazios ou zerados, é normal.
   * **ZHA:** Configurações → Dispositivos → o dispositivo → Controles / Sensores.
   * **Zigbee2MQTT:** o dispositivo → aba **Exposes**. As chaves do MQTT são sempre genéricas (`binary_output_4`, `analog_output_2`...), com o número do endpoint. O nome do ESPHome (`wifi_ota`, `ac_power`...) aparece como rótulo na aba Exposes e como nome da entidade no Home Assistant.
5. Se faltar alguma entidade depois de gravar um firmware novo, o coordenador ainda guarda os endpoints antigos. No Zigbee2MQTT use **Interview** e depois **Reconfigure**; no ZHA use **Reconfigurar**. Se não resolver, remova o dispositivo e pareie de novo.

### 5. Montar o divisor e os fios

Pinagem dos conectores, olhando pelas fotos dos guias. Nas fotos, **RX/TX estão nomeados do ponto de vista do ESP**:

| CN-CNT (5 pinos, trava para cima) | CN-WLAN (DNSK-P11) | Vai para a Super Mini |
| --- | --- | --- |
| 5V | 5V | pino **5V** |
| RX (sai do AC, 5 V) | RX (sai do AC, 5 V) | divisor → pino **RX (GPIO17)** |
| TX (entra no AC) | TX (entra no AC) | pino **TX (GPIO16)** |
| **12V — não ligar!** | — | nada (isole a ponta) |
| GND | Ground | pino **GND** |

1. Solde o resistor de 1 kΩ entre o fio **RX** do conector e o pino **RX (GPIO17)**.
2. Solde o resistor de 2 kΩ entre o pino **RX (GPIO17)** e o **GND**.
3. Ligue o fio **TX** do conector direto no pino **TX (GPIO16)**.
4. Ligue **5V** no pino **5V** e **GND** no **GND**.
5. No CN-CNT, isole o fio de **12 V** sozinho. Encostar 12 V em qualquer pino queima a placa.
6. Isole cada solda com termorretrátil.

Antes de ligar a Super Mini, com o AC energizado, meça no conector (ponta preta no GND):
* **5V**: cerca de 5 V
* **RX**: cerca de 5 V parado (é a saída do AC)
* **TX**: se der cerca de 5 V, o AC tem pull-up para 5 V; coloque um resistor de 1 kΩ em série entre esse fio e o GPIO16

### 6. Instalar no AC

1. **Desligue o disjuntor do AC.** Os contatos de rede ficam expostos.
2. Abra o AC e encontre o conector, seguindo o guia [CZ-TACG1](README.CZTACG1.md) ou [DNSK-P11](README.DNSKP11.md).
3. Desconecte o cabo USB da Super Mini e ligue os fios no conector do AC. Nunca use a USB e o 5 V do AC ao mesmo tempo.
4. Deixe a placa num lugar provisório onde dá para medir e mexer, e religue o disjuntor.
5. Só depois de tudo funcionar, acomode a placa no lugar do módulo Wi-Fi original (velcro ajuda) e feche o AC.

### 7. Conferir

Para o primeiro teste, vale gravar antes com `logger: level: VERBOSE`, que mostra cada pacote trocado com o AC (`TX: ...` e `RX: ...`). Depois volte para `DEBUG`.

1. Ligue o `wifi_ota` e abra **Logs → Wirelessly** no ESPHome.
   * CN-CNT: deve aparecer um `TX: 70 ...` (consulta) a cada 5 segundos, seguido de um `RX: ...` (resposta do AC).
   * CN-WLAN: aparecem os pacotes de handshake e, depois, as respostas.
2. Em até um minuto as entidades devem mostrar os valores reais do AC (modo, temperaturas etc.).
3. Teste nesta ordem, pelo Zigbee: `ac_target_temperature`, depois `ac_power`, depois `ac_mode` e `ac_fan`. Confira no próprio AC (visor e aletas) se o comando foi aceito.
4. Se só aparecer `TX` e nenhum `RX`:
   * confira se o `type` (`cnt` / `wlan`) está certo
   * inverta os fios RX e TX (é o erro mais comum)
   * meça o GPIO17: com o AC ligado deve ficar em ~3,3 V e oscilar quando o AC responde
5. Se aparecer `Dropping invalid packet`, o sinal chega mas com erro: confira o divisor e o GND.

### 8. Atualizações (OTA pelo Wi-Fi)

1. Ligue o switch `wifi_ota` pelo Zigbee.
2. Espere 10 a 20 segundos até o dispositivo aparecer online no ESPHome.
3. Clique em **Install → Wirelessly** (ou `esphome run ac-zigbee-esp32c6.yaml --device panasonic-ac-zigbee.local`).
4. Após a atualização a placa reinicia com o Wi-Fi desligado.

Se esquecer ligado, o Wi-Fi desliga sozinho em 30 minutos. Enquanto o Wi-Fi está ligado, o roteador Zigbee pode perder alguns pacotes.
