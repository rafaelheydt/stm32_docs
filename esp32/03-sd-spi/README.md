# ESP-IDF: cartão SD por SPI — guia de referência

## Resumo

Este guia monta um cartão SD no ESP32 por SPI e o expõe como diretório FAT (`/sdcard`), para gravar e ler arquivos com `fopen`, `fprintf` e `fclose`; de um projeto para outro mudam só a pinagem, o ponto de montagem e as opções de montagem.

| Item | Valor validado |
| --- | --- |
| Chip | ESP32 clássico, módulo WROOM-32 |
| ESP-IDF | v6.0.2 |
| Interface | Modo SPI do cartão (não SDMMC) |
| Sistema de arquivos | FAT via VFS, montado em `/sdcard` |
| Velocidade | Até 20 MHz: a documentação diz que o SD sobre SPI não passa de `SDMMC_FREQ_DEFAULT` |

Use SPI quando a gravação for esporádica ou moderada (logs, JSON Lines, fallback de dados) e você quiser liberdade para escolher os pinos. Para taxa alta e contínua, o modo SDMMC rende mais, mas só aceita os pinos dedicados do IO\_MUX.

O que está marcado como "validado" veio de um projeto real; o restante vem da documentação oficial listada em Fontes, no fim.

## Hardware: fiação, pull-ups e escolha de pinos

Quatro fios de sinal ligam o módulo ao ESP32, e as linhas do cartão exigem pull-ups de 10 kΩ, que as placas e módulos ESP32 comuns não trazem.

| Sinal do módulo | Função no modo SPI | Direção no ESP32 | Pino no projeto validado |
| --- | --- | --- | --- |
| MISO (DO) | Dados do cartão para o ESP32 | Entrada | GPIO 19 |
| MOSI (DI) | Dados do ESP32 para o cartão | Saída | GPIO 23 |
| SCK (CLK) | Clock | Saída | GPIO 18 |
| CS | Seleciona o cartão | Saída | GPIO 5 |
| VCC e GND | Alimentação e referência | n/a | Conforme o módulo; GND em comum com o ESP32 |

No padrão SD, em modo SPI o pino CMD faz o papel de MOSI, DAT0 o de MISO e DAT3 o de CS. O CS é controlado por software pelo driver.

### Pull-ups

Com o cartão em modo SPI, as linhas CMD e DAT0 a DAT3 devem ter pull-up de 10 kΩ. A página oficial de requisitos lista ESP32-DevKitC e módulos WROOM-32 como "sem pull-ups"; nesse caso, ligue cada linha a 3,3 V por um resistor de 10 kΩ ou use um módulo leitor que já os tenha (confira o esquema dele).

O campo `wait_for_miso` da configuração do dispositivo (padrão de 40 ms) existe para esperar o MISO subir antes de cada comando. A documentação diz que ele não deveria ser necessário com pull-ups corretos, então não o use para compensar a falta deles.

### Como escolher os pinos

- Qualquer pino de saída serve, porque no ESP32 o SPI pode passar pela GPIO Matrix. Os pinos do IO\_MUX do controlador têm melhor desempenho; os usados no projeto (18, 19, 23 e 5) são os padrão do SPI3/VSPI.
- Evite GPIO 6 a 11: são os pinos da flash.
- GPIO 34 a 39 só têm entrada, então servem apenas para MISO.
- Evite os pinos de strapping (0, 2, 12 e 15, e também o 5 com efeito menor) para sinais que fiquem em nível fixo no boot. O GPIO 12 é o mais delicado: define a tensão da flash em módulos com flash de 3,3 V.
- Se a placa já usa o SPI2 ou o SPI3 para outra coisa, use o outro controlador ou compartilhe o barramento (ver Desmontagem e vários dispositivos).

O cartão deve estar formatado em FAT32. O código não deve formatá-lo sozinho.

## Arquitetura

O acesso ao cartão atravessa seis camadas, e só a primeira é código do aplicativo: `fopen` passa pelo VFS e pelo FatFs, depois pelo driver SD/MMC, pelo host SD SPI e pelo driver SPI master, até chegar aos pinos.

&#91;embedded content: camadas do cartão SD · da chamada ao hardware\]

A função de montagem cobre as camadas 2 a 4; antes dela você só prepara o barramento SPI.

| Camada | Componente do ESP-IDF | Papel |
| --- | --- | --- |
| 1. Aplicação | stdio (`fopen`, `fprintf`, `fclose`) | Trata o cartão como um diretório comum |
| 2. VFS + FatFs | `esp_vfs_fat`, componente `fatfs` | Traduz o caminho `/sdcard/...` em arquivos e setores do volume FAT |
| 3. Driver SD/MMC | `sdmmc_cmd.h`, componente `sdmmc` | Inicializa o cartão e lê e escreve setores (`sdmmc_card_t`) |
| 4. Host SD SPI | `driver/sdspi_host.h`, componente `esp_driver_sdspi` | Fala o protocolo SD sobre SPI; o CS é controlado por software |
| 5. SPI master | `driver/spi_common.h`, componente `esp_driver_spi` | Gerencia o barramento (`spi_bus_initialize`), o DMA e o acesso entre tasks |
| 6. Pinos | GPIO Matrix ou IO\_MUX | Levam MOSI, MISO, SCK e CS ao cartão |

`esp_vfs_fat_sdspi_mount()` percorre as camadas 2 a 4 por você: anexa o cartão ao barramento, inicializa o cartão e monta o FAT. A única etapa anterior que você faz é inicializar o barramento (camada 5) com `spi_bus_initialize()`.

O SPI master permite dividir o mesmo barramento entre o cartão e outros dispositivos SPI, com restrições descritas na página oficial "Sharing the SPI Bus Among SD Cards and Other SPI Devices".

## Configuração

Quatro estruturas controlam o cartão (host, barramento, dispositivo e montagem), mais uma opção de sdkconfig e a lista de componentes no CMake.

### Estruturas

| Estrutura | Campo | Valor validado | O que significa e quando mudar |
| --- | --- | --- | --- |
| `sdmmc_host_t host = SDSPI_HOST_DEFAULT()` | `slot` | Padrão da macro | Identifica o controlador SPI (SPI2\_HOST ou SPI3\_HOST no ESP32). Atribua outro valor se o controlador já estiver em uso |
|  | `max_freq_khz` | 20000 (padrão) | Frequência máxima do clock. Reduza (por exemplo, 10000) se a fiação for instável |
| `spi_bus_config_t bus_cfg` | `mosi_io_num`, `miso_io_num`, `sclk_io_num` | 23, 19, 18 | Pinos do barramento |
|  | `quadwp_io_num`, `quadhd_io_num` | -1 | Pinos de modo quad, não usados |
|  | `max_transfer_sz` | 4000 | Maior transferência, em bytes; precisa cobrir ao menos um setor (512 bytes) |
| `spi_bus_initialize(host.slot, &bus_cfg, SDSPI_DEFAULT_DMA)` | 3º argumento | `SDSPI_DEFAULT_DMA` | Canal DMA da transferência |
| `sdspi_device_config_t slot_cfg = SDSPI_DEVICE_CONFIG_DEFAULT()` | `gpio_cs` | 5 | Pino de CS |
|  | `host_id` | `host.slot` | Precisa ser o mesmo controlador passado a `spi_bus_initialize` |
|  | `gpio_cd`, `gpio_wp` | Padrão (não usados) | Detecção de cartão e proteção contra escrita |
|  | `wait_for_miso` | Padrão (40 ms) | Espera pelo MISO alto; não use para compensar falta de pull-up |
| `esp_vfs_fat_sdmmc_mount_config_t mount_cfg` | `format_if_mount_failed` | `false` | Nunca formata sozinho, para não apagar um cartão com dados |
|  | `max_files` | 2 | Arquivos abertos ao mesmo tempo; cada um reserva RAM |
|  | `allocation_unit_size` | 16 KB | Unidade de alocação usada ao formatar; com `format_if_mount_failed = false` não deve ter efeito (confira no header da sua versão) |

Na documentação da v6.0 o tipo de montagem aparece como `esp_vfs_fat_mount_config_t`. O nome antigo (`esp_vfs_fat_sdmmc_mount_config_t`) compilou no projeto validado; em projeto novo, prefira o nome atual.

### sdkconfig

| Opção | Valores | Efeito |
| --- | --- | --- |
| `CONFIG_FATFS_LONG_FILENAMES` | `LFN_NONE`, `LFN_HEAP`, `LFN_STACK` | `LFN_NONE` limita os nomes ao formato 8.3: nome de até 8 caracteres e extensão de até 3, então `fallback.jsonl` falharia. Na documentação da v6.0 o padrão é `LFN_HEAP` (buffer na heap); em versões antigas o padrão era 8.3 |

Nenhuma outra opção de sdkconfig foi necessária no projeto validado.

### CMake

No ESP-IDF v6.0, declare todo componente cujo header você inclui:

```cmake
idf_component_register(SRCS "main.c" "sd_storage.c"
                    INCLUDE_DIRS "."
                    REQUIRES esp_driver_spi esp_driver_sdspi fatfs sdmmc)
```

O projeto validado compilou com `esp_driver_spi fatfs sdmmc` e `esp_driver_gpio`, sem `esp_driver_sdspi`. A documentação do host SD SPI, porém, indica `esp_driver_sdspi` para o header `driver/sdspi_host.h`, então declare-o para não depender de uma inclusão indireta.

Headers usados: `esp_vfs_fat.h`, `sdmmc_cmd.h`, `driver/sdspi_host.h` e `driver/spi_common.h`.

## Inicialização

Montar o cartão leva quatro passos em ordem fixa: perfil do host, barramento SPI, configuração do dispositivo e montagem do FAT.

1. `SDSPI_HOST_DEFAULT()` preenche o perfil padrão do host SD SPI.
2. `spi_bus_initialize()` liga o barramento nos pinos MOSI, MISO e SCK. Chame uma vez só.
3. `SDSPI_DEVICE_CONFIG_DEFAULT()` cria a configuração do dispositivo; você define `gpio_cs` e `host_id`.
4. `esp_vfs_fat_sdspi_mount()` anexa o cartão ao barramento, inicializa o cartão e monta o FAT no ponto de montagem.

```c
#include "esp_vfs_fat.h"
#include "sdmmc_cmd.h"
#include "driver/sdspi_host.h"
#include "driver/spi_common.h"

#define SD_MOUNT_POINT "/sdcard"
#define PIN_MISO 19
#define PIN_MOSI 23
#define PIN_SCK  18
#define PIN_CS   5

static sdmmc_card_t *s_card;

esp_err_t sd_init(void)
{
    sdmmc_host_t host = SDSPI_HOST_DEFAULT();

    spi_bus_config_t bus_cfg = {
        .mosi_io_num = PIN_MOSI,
        .miso_io_num = PIN_MISO,
        .sclk_io_num = PIN_SCK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 4000,
    };
    esp_err_t ret = spi_bus_initialize(host.slot, &bus_cfg, SDSPI_DEFAULT_DMA);
    if (ret != ESP_OK) return ret;

    sdspi_device_config_t slot_cfg = SDSPI_DEVICE_CONFIG_DEFAULT();
    slot_cfg.gpio_cs = PIN_CS;
    slot_cfg.host_id = host.slot;

    esp_vfs_fat_sdmmc_mount_config_t mount_cfg = {
        .format_if_mount_failed = false,
        .max_files = 2,
        .allocation_unit_size = 16 * 1024,
    };

    ret = esp_vfs_fat_sdspi_mount(SD_MOUNT_POINT, &host, &slot_cfg, &mount_cfg, &s_card);
    if (ret != ESP_OK) {
        spi_bus_free(host.slot);   // permite tentar de novo depois
        return ret;
    }

    sdmmc_card_print_info(stdout, s_card);
    return ESP_OK;
}
```

O `spi_bus_free()` na falha é um acréscimo recomendado: sem ele, uma segunda chamada de `sd_init()` falha em `spi_bus_initialize()` porque o barramento já foi inicializado. O projeto validado não o tinha porque nunca repetia a montagem.

### Como tratar o retorno

| Retorno de `esp_vfs_fat_sdspi_mount()` | Significado | Ação |
| --- | --- | --- |
| `ESP_OK` | Cartão inicializado e FAT montado | Usar `/sdcard` |
| `ESP_FAIL` | O cartão respondeu, mas o FAT não montou | Formatar em FAT32 no computador |
| Outro erro | O cartão não inicializou | Conferir fiação, pull-ups e alimentação |

Se o SD for opcional no produto, trate a falha com um aviso e siga o boot, guardando um flag de "cartão pronto" para o resto do código consultar.

## Uso do sistema de arquivos

Depois de montado, o cartão se usa com a biblioteca padrão de C, com o ponto de montagem como prefixo do caminho.

```c
// Acrescentar uma linha (cria o arquivo se não existir)
FILE *f = fopen("/sdcard/dados.jsonl", "a");
if (f) {
    fprintf(f, "%s\n", linha);
    fclose(f);                    // fecha e grava no cartão
}

// Ler linha a linha
FILE *r = fopen("/sdcard/dados.jsonl", "r");
if (r) {
    char buf[256];
    while (fgets(buf, sizeof(buf), r)) {
        // buf inclui o '\n' final
    }
    fclose(r);
}
```

| Prática | Motivo |
| --- | --- |
| Feche o arquivo (`fclose`) logo após gravar | Garante que os dados cheguem ao cartão; um reset com o arquivo aberto pode perder o que estava em buffer |
| Use um formato de uma linha por registro (JSON Lines, CSV) | Uma linha truncada por queda de energia não corrompe as anteriores |
| Dimensione o buffer de `fgets` acima da maior linha | Uma linha maior que o buffer é partida em duas leituras |
| Respeite `max_files` | Abrir mais arquivos que o configurado faz o `fopen` falhar |
| Serialize o acesso com um mutex se várias tasks gravarem | Evita linhas intercaladas; é boa prática, não uma exigência da documentação |
| Cheque o flag de "cartão pronto" antes de abrir | Evita erros em cascata quando o cartão não montou |

### Substituir um arquivo

O FAT não permite apagar uma linha do meio. Para manter só parte do conteúdo, grave o que sobra em um arquivo temporário, depois `remove` o original e `rename` o temporário para o nome original. A operação não é atômica: um reset entre `remove` e `rename` perde o conteúdo pendente.

### Nomes de arquivo

Com `LFN_HEAP` ou `LFN_STACK`, nomes longos funcionam, até 255 caracteres. Com `LFN_NONE`, use o formato 8.3, por exemplo `dados.txt` ou `log001.csv`.

## Desmontagem e vários dispositivos

Para liberar o cartão, desmonte o FAT primeiro e só depois libere o barramento; a ordem inversa deixa o sistema de arquivos usando um barramento que não existe mais.

```c
esp_vfs_fat_sdcard_unmount(SD_MOUNT_POINT, s_card);   // desmonta o FAT e libera o cartão
spi_bus_free(host.slot);                             // libera o barramento SPI
```

Confira o nome de `esp_vfs_fat_sdcard_unmount` em `esp_vfs_fat.h` na sua versão; ele não foi exercitado no projeto validado, que mantinha o cartão montado até o desligamento.

### Mais de um dispositivo no mesmo barramento

O driver SD SPI usa o SPI master, e a documentação diz que o barramento pode ser dividido entre cartões SD e outros dispositivos SPI, com o SPI master cuidando do acesso exclusivo entre tasks. Existem restrições, descritas na página oficial "Sharing the SPI Bus Among SD Cards and Other SPI Devices", que não foi consultada em detalhe para este guia.

- Inicialize o barramento uma vez (`spi_bus_initialize`) e anexe cada dispositivo com o próprio pino de CS.
- Cada dispositivo precisa de CS exclusivo; MOSI, MISO e SCK são comuns.
- Se houver outro dispositivo no barramento, use o `wait_for_miso` com cuidado: a documentação avisa para isso.

### Detecção de cartão

A estrutura do dispositivo tem `gpio_cd` (detecção de cartão) e `gpio_wp` (proteção contra escrita). Os valores padrão desligam os dois; este guia não os exercitou.

## Problemas comuns

A tabela leva cada sintoma de montagem ou de uso à causa mais provável e à ação que o resolve.

| Sintoma | Causa provável | Ação |
| --- | --- | --- |
| A montagem falha com erro diferente de `ESP_FAIL`, e o log pede para checar pull-ups | O cartão não inicializou: faltam pull-ups de 10 kΩ, fiação ruim ou alimentação fraca | Conferir os pull-ups em MOSI, MISO e CS, encurtar os fios, estabilizar os 3,3 V e reduzir `host.max_freq_khz` |
| A montagem retorna `ESP_FAIL` | O cartão respondeu, mas não há FAT válido | Formatar em FAT32 no computador; ligar `format_if_mount_failed` só se aceitar apagar o cartão |
| `fopen` devolve `NULL` com o cartão montado | Nome longo com `LFN_NONE`, ou limite de `max_files` atingido | Habilitar `CONFIG_FATFS_LONG_FILENAMES`, usar nome 8.3, ou fechar arquivos abertos |
| `spi_bus_initialize` falha na segunda tentativa de montar | O barramento já estava inicializado | Chamar `spi_bus_free()` antes de tentar de novo, ou inicializar o barramento uma única vez |
| O ESP32 não inicia com o módulo ligado | Um pino de strapping (0, 2, 12 ou 15) está sendo puxado no boot | Trocar o sinal para outro pino ou desconectar o módulo durante a gravação |
| Erros intermitentes de leitura ou escrita | Jumpers soltos, fios longos ou alimentação com queda | Reduzir `host.max_freq_khz`, encurtar fios e fixar os contatos |
| Linhas gravadas somem depois de um reset | O arquivo ficou aberto e os dados em buffer não chegaram ao cartão | Chamar `fclose` logo após gravar |
| O build falha com `driver/sdspi_host.h: No such file or directory` | O componente não está no `REQUIRES` | Adicionar `esp_driver_sdspi` (e conferir `esp_driver_spi`, `fatfs` e `sdmmc`) |

As duas primeiras linhas seguem a mensagem que o driver recomenda e a documentação de pull-ups. As demais combinam a documentação com boas práticas gerais de hardware, e algumas não foram reproduzidas no projeto validado.

## Checklist para reaproveitar

Em um projeto novo, percorra esta lista antes de escrever código; os itens 1 a 5 são decisões de hardware e de configuração, e os demais, de software.

- [ ] Confirmar o chip e a versão do ESP-IDF: nomes de componentes e de tipos de configuração mudam entre versões
- [ ] Escolher os quatro pinos, sem GPIO 6 a 11, sem pinos de strapping e com 34 a 39 só para MISO
- [ ] Verificar os pull-ups de 10 kΩ nas linhas do módulo
- [ ] Formatar o cartão em FAT32
- [ ] Decidir o controle de nomes de arquivo (`CONFIG_FATFS_LONG_FILENAMES`) conforme os nomes que o código usa
- [ ] Declarar `esp_driver_spi`, `esp_driver_sdspi`, `fatfs` e `sdmmc` no `REQUIRES`
- [ ] Inicializar na ordem: host, barramento, dispositivo, montagem, e conferir o retorno
- [ ] Decidir o que fazer se o cartão não montar (abortar ou seguir sem SD)
- [ ] Gravar com `fopen`, `fprintf` e `fclose`, e serializar o acesso entre tasks
- [ ] Se o código puder reiniciar a montagem, desmontar o FAT e liberar o barramento antes

### Fontes

- [SD Pull-up Requirements](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/sd_pullup_requirements.html): pull-ups de 10 kΩ em modo SPI e a lista de placas e módulos sem pull-ups.
- [SD SPI Host Driver](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/sdspi_host.html): arquitetura do host, limite de `SDMMC_FREQ_DEFAULT`, compartilhamento do barramento, `wait_for_miso` e o componente `esp_driver_sdspi`.
- [FAT Filesystem Support (v6.0)](https://docs.espressif.com/projects/esp-idf/en/release-v6.0/api-reference/storage/fatfs.html): opções de nomes longos; lido por trecho, não página inteira.
- [Sharing the SPI Bus Among SD Cards and Other SPI Devices](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/sdspi_share.html): indicada pela página do host, não aberta.
