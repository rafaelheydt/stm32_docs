# Microcontroladores — Tutoriais

Registro didático do aprendizado prático de microcontroladores, organizado por plataforma e, dentro de cada uma, em tutoriais sequenciais. Escopo atual: **STM32** e **ESP32**; aberto a outras plataformas no futuro.

## Plataformas

| Plataforma | Hardware em uso | Conteúdo |
| --- | --- | --- |
| [STM32](stm32/README.md) | STM32F103C8T6 "Blue Pill" + ST-Link V2 | Clock/debug, USART + HAL, driver Modbus RTU do PZEM-004T, integração FreeRTOS |
| [ESP32](esp32/README.md) | ESP32 (módulo DevKit) | Driver UART do ESP-IDF (fila de eventos), recepção dos dados do PZEM repassados pelo STM32 |

## Sobre o sistema completo

As duas plataformas fazem parte do mesmo sistema de leitura de energia: o **STM32** lê o sensor **PZEM-004T** via Modbus RTU e repassa os dados (JSON) por UART para o **ESP32**, que os recebe e (futuramente) publica. Projetos de firmware (fora deste repositório de docs):

- STM32: [`estudoSTM32`](..) (pasta raiz)
- ESP32: [`esp32_data_pzem`](../esp32_data_pzem)

<!--
## Convenção

Cada tutorial vive em sua própria pasta numerada, dentro da pasta da plataforma (`<plataforma>/0N-nome-da-tarefa/`), contendo:

- `README.md` — o passo a passo e as explicações daquela etapa
- `images/` — capturas de tela referenciadas no README

Ao concluir uma nova tarefa, criar a próxima pasta seguindo essa numeração e adicionar uma linha na tabela de tutoriais daquela plataforma (`<plataforma>/README.md`).
-->
