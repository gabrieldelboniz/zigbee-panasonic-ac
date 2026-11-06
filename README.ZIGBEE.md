# Panasonic AC via Zigbee (ESP32-C6 Super Mini)

Configuração de exemplo: [`ac-zigbee-esp32c6.yaml.example`](ac-zigbee-esp32c6.yaml.example)

O ESP32-C6 controla o AC pela UART e ao mesmo tempo funciona como roteador (repetidor) Zigbee. O Wi-Fi fica desligado e só é ligado, por um switch Zigbee, para atualizações OTA.

## Requisitos

* ESP32-C6 Super Mini (ou outra placa ESP32-C6/H2, ajustando os pinos)
* ESPHome com Zigbee nativo para ESP32 (2026.5 ou mais novo). Não use o componente externo `luar123/zigbee_esphome`.
* Coordenador Zigbee: ZHA ou Zigbee2MQTT

> O nRF52840 não serve para o AC hoje: o ESPHome ainda não tem driver de UART para o nRF52. Ele pode ser usado como roteador Zigbee.

## Ligação

| AC | ESP32-C6 |
| --- | --- |
| TX (5 V) | divisor de tensão → pino RX (GPIO17) |
| RX | pino TX (GPIO16), direto (o AC aceita 3,3 V) |
| 5 V | 5V |
| GND | GND |

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

## Atualização (OTA)

1. Ligue o `wifi_ota` pelo Zigbee.
2. Espere 10 a 20 segundos e faça o OTA pelo dashboard do ESPHome.
3. Após a atualização a placa reinicia com o Wi-Fi desligado.

Se esquecer ligado, o Wi-Fi desliga sozinho em 30 minutos. Enquanto o Wi-Fi está ligado, o roteador Zigbee pode perder alguns pacotes.

A primeira gravação é pela USB. Crie um `secrets.yaml` com `wifi_ssid` e `wifi_password`.
