# STM32 + FreeRTOS — Tutoriais

Registro didático do aprendizado prático de STM32 + FreeRTOS, organizado em tutoriais sequenciais.

**Hardware em uso:** STM32F103C8T6 "Blue Pill" + ST-Link V2
**Ambiente:** STM32CubeMX + VS Code (CMake) ou STM32CubeIDE

## Tutoriais

| # | Tutorial | Categoria | Conteúdo |
| --- | --- | --- | --- |
| 01 | [Introdução](01-introducao/README.md) | Periférico (Clock/Debug) | Configuração inicial do projeto: clock (HSE + PLL + barramentos) e debug (Serial Wire) |
| 02 | [USART](02-usart/README.md) | Periférico | Configuração do USART3 e funções da HAL (`HAL_UART_Transmit`/`HAL_UART_Receive`) |

## Projetos

| Projeto | Conteúdo |
| --- | --- |
| [PZEM-004T](pzem/README.md) | Driver Modbus RTU + integração FreeRTOS para leitura do sensor de energia PZEM-004T |

<!--
## Convenção

Cada tutorial vive em sua própria pasta numerada (`0N-nome-da-tarefa/`), contendo:

- `README.md` — o passo a passo e as explicações daquela etapa
- `images/` — capturas de tela referenciadas no README

Ao concluir uma nova tarefa, criar a próxima pasta seguindo essa numeração (ex.: `02-usart/`) e adicionar uma linha na tabela acima.
-->
