# ESP-IDF: Wi-Fi manager em modo estação — guia de referência

## Resumo

Este guia monta uma conexão Wi-Fi em modo estação no ESP32, com credenciais no `menuconfig` e reconexão automática, em um módulo (`wifi_manager`) que o resto do programa usa com duas chamadas: conectar e perguntar se há rede.

| Item | Valor validado |
| --- | --- |
| Chip | ESP32 clássico (módulo WROOM-32), só 2,4 GHz |
| ESP-IDF | v6.0.2 |
| Modo | Estação (STA) com DHCP |
| Segurança mínima aceita | WPA2-PSK |
| Reconexão | Tentativas por rodada, mais uma nova rodada agendada por timer |

O módulo expõe três itens:

- `wifi_manager_connect()`: inicializa e conecta; bloqueia até conectar ou a primeira rodada de tentativas falhar.
- `wifi_manager_is_connected()`: estado atual da conexão, sem bloquear.
- Um event group com `WIFI_CONNECTED_BIT` e `WIFI_FAIL_BIT`, para quem precisa esperar pela rede.

De um projeto para outro mudam as credenciais, o número de tentativas e o intervalo entre rodadas; hostname, IP fixo e economia de energia são ajustes opcionais.

O que está marcado como "validado" veio de um projeto real; o restante vem da documentação oficial listada em Fontes, no fim.

## Pré-requisitos

Quatro coisas precisam existir antes do código do Wi-Fi: as opções no Kconfig, a lista de componentes no CMake, a NVS inicializada e um único event loop padrão.

### Kconfig

Crie `main/Kconfig.projbuild`; as opções aparecem no `menuconfig` e viram macros `CONFIG_...` no código.

```
menu "Wi-Fi Manager"

    config WIFI_SSID
        string "Wi-Fi SSID"
        default "minha_rede"

    config WIFI_PASSWORD
        string "Wi-Fi Password"
        default "minha_senha"

    config WIFI_MAXIMUM_RETRY
        int "Máximo de tentativas por rodada"
        default 5

    config WIFI_RECONNECT_INTERVAL_S
        int "Intervalo entre rodadas de reconexão (segundos)"
        default 30
        range 5 600

endmenu
```

Preencha SSID e senha em `idf.py menuconfig`. Os valores ficam no `sdkconfig` em texto puro, então não versione esse arquivo com credenciais reais; mantenha um `sdkconfig.defaults` sem segredos. Se o projeto tiver outros componentes com Kconfig, prefixe os nomes para evitar colisão.

### CMake

```cmake
idf_component_register(SRCS "main.c" "wifi_manager.c"
                    INCLUDE_DIRS "."
                    REQUIRES esp_wifi esp_event esp_netif nvs_flash esp_timer)
```

| Componente | Por que precisa |
| --- | --- |
| `esp_wifi` | Driver Wi-Fi (`esp_wifi.h`) |
| `esp_event` | Event loop e registro de handlers |
| `esp_netif` | Interface de rede e cliente DHCP |
| `nvs_flash` | NVS, usada pelo driver Wi-Fi e inicializada no `app_main` |
| `esp_timer` | Timer da nova rodada de reconexão |

No ESP-IDF v6.0, declare todo componente cujo header você inclui, mesmo que outro componente o puxe por dependência indireta.

### NVS

`esp_wifi_init()` usa a partição NVS, então `nvs_flash_init()` roda antes do Wi-Fi. Se retornar `ESP_ERR_NVS_NO_FREE_PAGES` ou `ESP_ERR_NVS_NEW_VERSION_FOUND`, a partição está cheia ou foi escrita por outra versão da biblioteca; o padrão é apagá-la e inicializar de novo.

```c
esp_err_t ret = nvs_flash_init();
if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
}
ESP_ERROR_CHECK(ret);
```

O erase apaga tudo que houver na NVS. Se o projeto guardar outros dados ali, esse tratamento precisa mudar.

### Event loop padrão

`esp_event_loop_create_default()` só pode ser chamado uma vez no programa. Se outro módulo já o criou, a segunda chamada retorna `ESP_ERR_INVALID_STATE`; nesse caso, crie o loop uma vez no `app_main` e remova a chamada do `wifi_manager`.

## Arquitetura

A conexão é um ciclo de sete partes: a aplicação pede, o driver trabalha em segundo plano, e quem libera a aplicação é um evento tratado pelo handler, não o retorno de uma chamada.

&#91;embedded content: partes do Wi-Fi manager · ciclo de uma conexão\]

O ciclo corre no sentido horário: a aplicação pede, o driver emite eventos, o handler liga um bit e a aplicação é liberada; o timer volta ao handler quando uma rodada de tentativas termina.

| Parte | Papel |
| --- | --- |
| Aplicação | Chama `wifi_manager_connect()` e espera; depois consulta `wifi_manager_is_connected()` |
| `wifi_manager` | Configura o driver, registra o handler e guarda o estado (contador de tentativas, flag de conexão) |
| Driver Wi-Fi e `esp_netif` | O rádio associa ao roteador; o cliente DHCP obtém o IP; o driver usa a NVS |
| Event loop padrão (`esp_event`) | Entrega os eventos do Wi-Fi e do IP ao handler, em uma task própria |
| `event_handler` | Função do `wifi_manager` que reage a cada evento e avança o ciclo |
| Event group (FreeRTOS) | Variável de bits que o handler liga e a aplicação espera |
| `esp_timer` | Agenda uma nova rodada de reconexão quando as tentativas se esgotam |

Como o handler roda na task do event loop e a aplicação roda na sua própria task, os dois se comunicam pelo event group e pela flag `connected`, que por isso é `volatile`.

## Inicialização

A função de conexão segue uma ordem fixa em que os handlers são registrados antes de ligar o driver, para não perder o primeiro evento.

1. Criar o event group.
2. Criar o timer de reconexão.
3. `esp_netif_init()` e `esp_event_loop_create_default()`.
4. `esp_netif_create_default_wifi_sta()`.
5. `esp_wifi_init()` com `WIFI_INIT_CONFIG_DEFAULT()`.
6. Registrar o handler para `WIFI_EVENT` (qualquer id) e para `IP_EVENT_STA_GOT_IP`.
7. `esp_wifi_set_mode()` e `esp_wifi_set_config()` com SSID, senha e autenticação mínima.
8. `esp_wifi_start()`.
9. Esperar o event group por `CONNECTED` ou `FAIL`.

```c
bool wifi_manager_connect(void)
{
    s_wifi_event_group = xEventGroupCreate();

    const esp_timer_create_args_t timer_args = {
        .callback = reconnect_timer_cb,
        .name = "wifi_reconnect",
    };
    ESP_ERROR_CHECK(esp_timer_create(&timer_args, &s_reconnect_timer));

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t init_cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&init_cfg));

    // O último argumento (instância) pode ser NULL se você não for cancelar o registro
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL, NULL));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL, NULL));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = CONFIG_WIFI_SSID,
            .password = CONFIG_WIFI_PASSWORD,
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
        },
    };
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    EventBits_t bits = xEventGroupWaitBits(
        s_wifi_event_group,
        WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
        pdFALSE,          // não limpa os bits ao sair
        pdFALSE,          // basta um dos dois
        portMAX_DELAY);

    return (bits & WIFI_CONNECTED_BIT) != 0;
}
```

O código usa `ESP_ERROR_CHECK`, que aborta o programa em qualquer erro; num produto, troque por tratamento de erro onde a falha for recuperável.

### Autenticação mínima

`.threshold.authmode` define o nível de segurança mínimo aceito. Com `WIFI_AUTH_WPA2_PSK`, o ESP32 só conecta em redes WPA2 ou mais fortes, e rejeita uma rede aberta com o mesmo nome. A documentação descreve o campo assim: só entram na escolha os pontos de acesso com modo de autenticação mais seguro que o escolhido e sinal acima do RSSI mínimo.

### Variáveis do módulo

```c
static EventGroupHandle_t s_wifi_event_group;
static esp_timer_handle_t s_reconnect_timer;
static volatile int  s_retry_count = 0;     // falhas consecutivas da rodada atual
static volatile bool s_connected  = false;  // estado atual, lido por outras tasks
```

## Eventos e event group

O handler trata quatro eventos, e o sucesso da conexão é o `GOT_IP`, não a associação ao roteador: sem IP não há como usar a rede.

&#91;embedded content: ciclo de vida do Wi-Fi · sucesso e reconexão\]

O caminho de cima leva ao `GOT_IP`; qualquer falha cai em `STA_DISCONNECTED` e, enquanto houver tentativas, volta a `esp_wifi_connect()`.

| Evento | Quando acontece | O que o handler faz |
| --- | --- | --- |
| `WIFI_EVENT_STA_START` | O driver foi ligado | Chama `esp_wifi_connect()` |
| `WIFI_EVENT_STA_CONNECTED` | O link de rádio associou ao roteador | Nada (ignorado) |
| `IP_EVENT_STA_GOT_IP` | O DHCP entregou um IP | Zera o contador, marca `connected`, cancela o timer, limpa `FAIL_BIT` e liga `CONNECTED_BIT` |
| `WIFI_EVENT_STA_DISCONNECTED` | O link caiu ou a tentativa falhou | Marca `connected = false`, limpa `CONNECTED_BIT` e decide entre reconectar e esgotar a rodada |

```c
static void event_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (base == WIFI_EVENT && id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (base == WIFI_EVENT && id == WIFI_EVENT_STA_DISCONNECTED) {
        s_connected = false;
        xEventGroupClearBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
        // reconectar ou esgotar a rodada: ver "Reconexão automática"
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *e = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "IP obtido: " IPSTR, IP2STR(&e->ip_info.ip));

        s_retry_count = 0;
        s_connected = true;
        esp_timer_stop(s_reconnect_timer);   // retorna erro se o timer não estava ativo; ignorável
        xEventGroupClearBits(s_wifi_event_group, WIFI_FAIL_BIT);
        xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
    }
}
```

### Event group

Um event group é uma variável compartilhada em que cada bit sinaliza um estado: `BIT0` é `WIFI_CONNECTED_BIT` e `BIT1` é `WIFI_FAIL_BIT`. Quem espera fica suspenso, sem gastar CPU, até outra task ligar o bit.

| Chamada | Uso no módulo |
| --- | --- |
| `xEventGroupSetBits` | O handler liga `CONNECTED` (no `GOT_IP`) ou `FAIL` (ao esgotar a rodada) |
| `xEventGroupClearBits` | O handler desliga `CONNECTED` na queda e `FAIL` no sucesso, para os dois bits ficarem mutuamente exclusivos |
| `xEventGroupWaitBits` | A aplicação espera por `CONNECTED \| FAIL`; o primeiro `pdFALSE` mantém o bit ligado ao sair e o segundo aceita um bit só |

Os bits sinalizam transições. Para saber se há rede neste instante, consulte `wifi_manager_is_connected()`, que lê a flag `connected`.

## Reconexão automática

A reconexão tem dois níveis: tentativas imediatas dentro de uma rodada e, quando elas se esgotam, um timer que agenda a próxima rodada, de modo que o Wi-Fi volta sozinho sem reset.

1. `STA_DISCONNECTED` limpa o `CONNECTED_BIT` e marca `connected = false`.
2. Se `s_retry_count < CONFIG_WIFI_MAXIMUM_RETRY`, chama `esp_wifi_connect()` e incrementa o contador.
3. Senão, liga o `FAIL_BIT` e inicia o `esp_timer` one-shot com `CONFIG_WIFI_RECONNECT_INTERVAL_S` segundos.
4. Quando o timer dispara, zera o contador, limpa o `FAIL_BIT` e chama `esp_wifi_connect()`. Se falhar de novo, volta ao passo 1.
5. `GOT_IP` cancela o timer, zera o contador e liga o `CONNECTED_BIT`.

```c
// Dentro do ramo STA_DISCONNECTED do event_handler:
if (s_retry_count < CONFIG_WIFI_MAXIMUM_RETRY) {
    esp_wifi_connect();
    s_retry_count++;
} else {
    xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
    if (!esp_timer_is_active(s_reconnect_timer)) {
        esp_timer_start_once(s_reconnect_timer,
            (uint64_t)CONFIG_WIFI_RECONNECT_INTERVAL_S * 1000000ULL);
    }
}

static void reconnect_timer_cb(void *arg)
{
    s_retry_count = 0;
    xEventGroupClearBits(s_wifi_event_group, WIFI_FAIL_BIT);
    esp_wifi_connect();
}
```

### Regras que o código precisa manter

| Regra | Por que |
| --- | --- |
| `CONNECTED_BIT` e `FAIL_BIT` nunca ficam ligados juntos | Estados contraditórios enganam quem consulta o event group |
| O `CONNECTED_BIT` é limpo na queda | Um bit antigo ligado faz a aplicação achar que há rede |
| `s_retry_count` zera no `GOT_IP` e a cada nova rodada | O contador mede falhas consecutivas, não falhas totais |
| `s_retry_count` e `s_connected` são `volatile` | São escritos pela task do event loop e do timer e lidos por outras tasks |
| O timer é criado antes de `esp_wifi_start()` | O handler pode precisar dele logo nas primeiras falhas |
| `wifi_manager_connect()` retornar `false` não encerra as tentativas | Só indica que a primeira rodada falhou; use `wifi_manager_is_connected()` para saber o estado atual |

O driver tem a sua própria contagem de tentativas (`failure_retry_cnt` em `wifi_sta_config_t`, repetições antes de passar ao próximo ponto de acesso), independente do contador do módulo.

### Ajuste dos tempos

O número de tentativas e o intervalo são configuráveis no `menuconfig`. Um intervalo curto reconecta mais rápido depois de um reinício do roteador, ao custo de mais tentativas; um longo poupa energia e o roteador. Os padrões do projeto validado são 5 tentativas e 30 s, e a reconexão por timer não foi exercitada em condição real de queda.

## Opções para adaptar

Seis ajustes cobrem a maioria das adaptações: hostname, IP fixo, segurança, economia de energia, ponto de acesso fixo e desligamento. Só o IP fixo, o padrão de economia de energia e os campos de `wifi_sta_config_t` foram conferidos na documentação; o restante deve ser confirmado nos headers da sua versão.

| Necessidade | Como fazer | Observação |
| --- | --- | --- |
| Hostname | `esp_netif_set_hostname(netif, "nome")` | Guarde o ponteiro devolvido por `esp_netif_create_default_wifi_sta()` e chame antes de iniciar o Wi-Fi, para o DHCP já usar o nome |
| IP fixo | `esp_netif_dhcpc_stop(netif)`, preencher `esp_netif_ip_info_t` (ip, netmask, gw) e chamar `esp_netif_set_ip_info(netif, &info)` | Receita da FAQ da Espressif; faça antes de `esp_wifi_start()`. Para DNS, veja `esp_netif_set_dns_info` |
| Segurança mínima | `.threshold.authmode` | `WIFI_AUTH_WPA2_PSK` aceita WPA2 ou mais forte. Para WPA3, os campos `sae_pwe_h2e` e `pmf_cfg` entram; o exemplo oficial de estação mostra a combinação |
| Economia de energia | `esp_wifi_set_ps(tipo)` | O padrão é `WIFI_PS_MIN_MODEM`; `WIFI_PS_NONE` reduz a latência e aumenta o consumo |
| Prender a um ponto de acesso ou canal | `channel` (0 se desconhecido), `bssid_set` e `bssid` | Útil com vários roteadores do mesmo nome; `scan_method` e `sort_method` controlam a escolha entre eles |
| Desligar o Wi-Fi | `esp_wifi_disconnect()`, `esp_wifi_stop()` e `esp_wifi_deinit()` | Cancele o timer e libere o event group e os handlers criados pelo módulo |

Exemplo de IP fixo, a inserir entre `esp_netif_create_default_wifi_sta()` e `esp_wifi_start()`:

```c
esp_netif_t *netif = esp_netif_create_default_wifi_sta();

esp_netif_dhcpc_stop(netif);
esp_netif_ip_info_t info = {0};
info.ip.addr      = ESP_IP4TOADDR(192, 168, 1, 50);
info.netmask.addr = ESP_IP4TOADDR(255, 255, 255, 0);
info.gw.addr      = ESP_IP4TOADDR(192, 168, 1, 1);
esp_netif_set_ip_info(netif, &info);
```

Com IP fixo, confirme em teste que o `IP_EVENT_STA_GOT_IP` ainda chega ao handler. Se não chegar, trate `WIFI_EVENT_STA_CONNECTED` como o momento de marcar a conexão como pronta. Este guia não exercitou essa variação.

## Problemas comuns

Cada sintoma abaixo aponta para uma causa provável e para a ação que o resolve.

| Sintoma | Causa provável | Ação |
| --- | --- | --- |
| O programa aborta em `esp_wifi_init()` logo no boot | A NVS não foi inicializada | Chamar `nvs_flash_init()` antes do Wi-Fi, com erase e novo init se vier `ESP_ERR_NVS_NO_FREE_PAGES` ou `ESP_ERR_NVS_NEW_VERSION_FOUND` |
| `esp_event_loop_create_default()` retorna `ESP_ERR_INVALID_STATE` | Outro módulo já criou o loop padrão | Criar o loop uma vez no `app_main` e remover do `wifi_manager` |
| Só chegam eventos `STA_DISCONNECTED` | SSID ou senha errados, rede de 5 GHz num ESP32 de 2,4 GHz, ou segurança abaixo do mínimo configurado | Conferir as credenciais no `menuconfig`, usar uma rede de 2,4 GHz e revisar `.threshold.authmode` |
| Associa ao roteador, mas o `GOT_IP` não chega | O DHCP não respondeu | Conferir o roteador (faixa de IPs livres) ou configurar IP fixo |
| A aplicação acha que há rede depois de uma queda | O `CONNECTED_BIT` não foi limpo no `STA_DISCONNECTED` | Limpar o bit na queda e usar `wifi_manager_is_connected()` para o estado atual |
| `wifi_manager_connect()` retorna `false` e o Wi-Fi nunca volta | Não há nova rodada de reconexão | Agendar a nova rodada com o `esp_timer`, como em Reconexão automática |
| O resto do programa só inicia depois do Wi-Fi | `wifi_manager_connect()` bloqueia | Iniciar as partes críticas antes de chamá-lo, ou chamá-lo em uma task própria |
| Pacotes chegam com atraso | A economia de energia padrão (`WIFI_PS_MIN_MODEM`) | Testar `esp_wifi_set_ps(WIFI_PS_NONE)`, aceitando o maior consumo |
| O build falha com `esp_wifi.h`, `nvs_flash.h` ou `esp_timer.h` não encontrado | O componente não está no `REQUIRES` | Adicionar `esp_wifi`, `nvs_flash` e `esp_timer` (e `esp_event`, `esp_netif`) |

As linhas da NVS, do event loop e do `REQUIRES` seguem o comportamento das APIs. As linhas do `CONNECTED_BIT` e da nova rodada vieram da revisão do código do projeto de origem e foram corrigidas lá, mas não reproduzidas como falha em campo.

## Checklist para reaproveitar

Em um projeto novo, percorra esta lista antes de testar a conexão; os quatro primeiros itens são de configuração e os demais, de comportamento.

- [ ] Confirmar o chip (banda de 2,4 GHz ou 5 GHz) e a versão do ESP-IDF
- [ ] Criar `main/Kconfig.projbuild` com SSID, senha, tentativas e intervalo, e não versionar o `sdkconfig` com credenciais
- [ ] Declarar `esp_wifi`, `esp_event`, `esp_netif`, `nvs_flash` e `esp_timer` no `REQUIRES`
- [ ] Inicializar a NVS no `app_main`, antes do Wi-Fi
- [ ] Garantir que o event loop padrão seja criado uma única vez no programa
- [ ] Registrar os handlers antes de `esp_wifi_start()`
- [ ] Tratar o sucesso no `GOT_IP`, não na associação ao roteador
- [ ] Limpar o `CONNECTED_BIT` na queda e manter os dois bits mutuamente exclusivos
- [ ] Agendar uma nova rodada de reconexão com `esp_timer`
- [ ] Iniciar o que for crítico antes do `wifi_manager_connect()` bloqueante
- [ ] Testar derrubando o roteador por alguns minutos e conferindo as rodadas e o retorno do IP

### Fontes

- [ESP-IDF Wi-Fi API (v5.0.5)](https://docs.espressif.com/projects/esp-idf/en/v5.0.5/api-reference/network/esp_wifi.html): modo estação como padrão e tipo de economia de energia padrão (`WIFI_PS_MIN_MODEM`).
- [Wi-Fi FAQ da Espressif](https://docs.espressif.com/projects/esp-faq/en/latest/software-framework/wifi.html): receita de IP fixo em modo estação (`esp_netif_dhcpc_stop` e `esp_netif_set_ip_info`).
- [Campos de `wifi_sta_config_t`](https://sourcevu.sysprogs.com/espressif/esp-idf/symbols/wifi_sta_config_t): índice de terceiros do código do ESP-IDF; descrição de `threshold`, `channel`, `scan_method` e `failure_retry_cnt`.
