# ESP-IDF: envio HTTP com fallback local — guia de referência

Oct 5, 2026 · @Rafael

## Resumo

Este guia monta, no ESP32, o padrão "envia agora; se não der, guarda localmente e reenvia depois": uma fila recebe os dados, uma task tenta o envio por HTTP, e um disjuntor simples evita que um servidor fora do ar prenda a task em timeouts repetidos.

| Item | Valor validado |
| --- | --- |
| Chip | ESP32 clássico (módulo WROOM-32) |
| ESP-IDF | v6.0.2 |
| Cliente HTTP | `esp_http_client`, um `POST` bloqueante por envio |
| Armazenamento local | Arquivo em cartão SD, uma linha por registro (ver [o guia de cartão SD por SPI](../03-sd-spi/README.md)) |

Use este padrão quando o envio pode falhar por período curto ou longo (rede instável, servidor fora do ar) e perder dados não é aceitável. Para um fluxo que só precisa de "tentar uma vez, descartar se falhar", a função de envio sozinha já basta, sem fila nem fallback.

De um projeto para outro mudam o formato dos dados, o destino do fallback (SD, NVS, nada) e os tempos (timeout, duração do disjuntor, tamanho da fila); a estrutura — fila, disjuntor, reenvio — se mantém.

O que está marcado como "validado" veio de um projeto real, testado em cenários simulados no PC; o restante vem da documentação oficial listada em Fontes, no fim.

## Pré-requisitos

### CMake

```cmake
idf_component_register(SRCS "main.c" "http_client.c" "data_sender.c"
                    INCLUDE_DIRS "."
                    REQUIRES esp_http_client)
```

Se o fallback usa Wi-Fi e SD (como no projeto validado), acrescente os componentes desses módulos; veja os guias de SD por SPI e de Wi-Fi manager, cada um com o seu `REQUIRES`.

### As três funções que o padrão precisa receber de fora

O `data_sender` em si não sabe nada de HTTP nem de SD: ele só coordena fila, disjuntor e reenvio em torno de três funções, que podem vir de qualquer módulo.

| Função | Assinatura | Papel |
| --- | --- | --- |
| Verificar se há rede | `bool algo_esta_pronto(void)` | Decide se vale tentar o envio agora, sem pagar o timeout de uma tentativa fadada a falhar |
| Enviar um item | `bool enviar(const char *dado)` | Faz a operação de rede; `true` só com sucesso confirmado (por exemplo, HTTP 2xx) |
| Gravar e reenviar o fallback | `bool gravar(const char *dado)` e `bool reenviar(bool (*send_fn)(const char *))` | Guardam o que não foi enviado e o devolvem mais tarde, usando a mesma função de envio |

Essa separação é o que permite testar a lógica da fila e do disjuntor no PC, sem ESP-IDF: basta substituir as três funções por versões de teste com a mesma assinatura (ver Problemas comuns para um exemplo).

## Arquitetura

Uma task consome uma fila e decide, item a item, entre enviar e gravar localmente; a mesma task também reenvia o que ficou pendente quando o destino volta a responder.

&#91;embedded content: fila + disjuntor · decisão a cada item\]

| Parte | Papel |
| --- | --- |
| Fila (FreeRTOS) | Desacopla quem produz os itens de quem os envia; quem produz nunca bloqueia |
| `pode_tentar()` | Combina "há rede/canal disponível" com "o disjuntor está fechado" |
| `enviar()` | Uma operação de rede bloqueante, com timeout próprio |
| Disjuntor | Um instante futuro (`esp_timer_get_time() + N segundos`) até o qual `enviar()` nem é tentado |
| Armazenamento local | Grava o que não foi entregue e reenvia depois, pela mesma função de envio |

O disjuntor existe para o caso de falha persistente, não de falha isolada: sem ele, um destino fora do ar por minutos faz a task pagar um timeout a cada item, até a fila do FreeRTOS encher e os itens mais novos serem descartados antes mesmo de uma tentativa.

## Código mínimo

### Função de envio genérica

```c
bool enviar_http(const char *payload)
{
  if (!rede_disponivel()) {
    return false;   // falha rápida, sem pagar o timeout
  }

  esp_http_client_config_t config = {
    .url = CONFIG_ENDPOINT_URL,
    .method = HTTP_METHOD_POST,
    .timeout_ms = 5000,
  };
  esp_http_client_handle_t client = esp_http_client_init(&config);
  if (client == NULL) return false;

  esp_http_client_set_header(client, "Content-Type", "application/json");
  esp_http_client_set_post_field(client, payload, (int)strlen(payload));

  bool ok = false;
  if (esp_http_client_perform(client) == ESP_OK) {
    int status = esp_http_client_get_status_code(client);
    ok = (status >= 200 && status < 300);
  }

  esp_http_client_cleanup(client);
  return ok;
}
```

`rede_disponivel()` depende do transporte (`wifi_connected()`, um estado de modem celular, e assim por diante); o resto da função não muda entre projetos.

### Task com fila e disjuntor

```c
typedef struct { char payload[TAMANHO_MAX]; } item_t;

static QueueHandle_t s_fila;
static bool s_pendente = false;
static int64_t s_disjuntor_ate_us = 0;

static bool pode_tentar(void)
{
  return rede_disponivel() && esp_timer_get_time() >= s_disjuntor_ate_us;
}

static void abrir_disjuntor(int segundos)
{
  s_disjuntor_ate_us = esp_timer_get_time() + (int64_t)segundos * 1000000LL;
}

static void entregar(const char *payload)
{
  if (pode_tentar()) {
    if (enviar_http(payload)) return;     // sucesso: nada mais a fazer
    abrir_disjuntor(30);
  }
  if (gravar_fallback(payload)) {
    s_pendente = true;
  }
}

static void task_envio(void *arg)
{
  item_t item;
  s_pendente = ha_pendencia_no_fallback();

  for (;;) {
    if (xQueueReceive(s_fila, &item, pdMS_TO_TICKS(1000)) == pdTRUE) {
      entregar(item.payload);
    }
    if (s_pendente && pode_tentar()) {
      reenviar_fallback(enviar_http);
      s_pendente = ha_pendencia_no_fallback();
      if (s_pendente) abrir_disjuntor(30);
    }
  }
}
```

O timeout de 1 s no `xQueueReceive` não é para o item em si: é o que garante que a tentativa de reenvio aconteça mesmo em períodos sem itens novos, para o disjuntor reabrir sozinho quando a janela vencer.

Quem produz os itens nunca bloqueia: `xQueueSend(s_fila, &item, 0)`, com timeout 0 — fila cheia ou item grande demais é descartado e logado, não travado.

## Reenvio de pendências

Um sistema de arquivos comum não remove uma linha do meio, então o reenvio lê o arquivo original, copia para um temporário só o que ainda não foi enviado, e troca os dois no final.

```c
bool reenviar_fallback(bool (*enviar)(const char *))
{
  FILE *origem = fopen(CAMINHO_FALLBACK, "r");
  if (origem == NULL) return false;   // nada para reenviar

  FILE *tmp = fopen(CAMINHO_TMP, "w");
  if (tmp == NULL) { fclose(origem); return false; }

  char linha[TAMANHO_MAX];
  bool parou = false;

  while (fgets(linha, sizeof(linha), origem) != NULL) {
    size_t n = strlen(linha);
    if (n > 0 && linha[n - 1] == '\n') linha[n - 1] = '\0';

    if (!parou && enviar(linha)) {
      continue;          // sucesso: a linha não é copiada, "desaparece"
    }
    parou = true;          // a primeira falha interrompe as tentativas
    fprintf(tmp, "%s\n", linha);
  }

  fclose(origem);
  fclose(tmp);
  remove(CAMINHO_FALLBACK);
  rename(CAMINHO_TMP, CAMINHO_FALLBACK);
  return true;
}
```

### Por que parar na primeira falha

Sem o `parou`, o reenvio tentaria enviar todas as linhas, mesmo depois de uma falhar — com o destino fora do ar, isso paga um timeout por linha, podendo levar minutos para um arquivo com poucas dezenas de registros. Parar cedo assume que, se a entrega de uma linha falhou, as seguintes têm boa chance de falhar pelo mesmo motivo (rede ainda fora, servidor ainda rejeitando); a próxima tentativa fica para a volta seguinte do loop, depois do disjuntor reabrir.

### O que essa troca de arquivo não garante

Um reset do dispositivo entre o `remove` e o `rename` perde as linhas que estavam pendentes naquele instante: a operação não é atômica. Para a maioria dos usos esse risco é aceitável; onde não for, grave num nome temporário por linha ou use um formato que suporte transação (fora do escopo deste guia).

## Configuração

| Parâmetro | Em quê | Como escolher |
| --- | --- | --- |
| Timeout do `enviar()` | `esp_http_client_config_t.timeout_ms` | Tempo que uma task fica presa por tentativa; alto demais atrasa a fila, baixo demais rejeita um destino só um pouco lento |
| Duração do disjuntor | `abrir_disjuntor(segundos)` | Curta demais insiste contra um destino ainda fora do ar; longa demais atrasa a recuperação quando ele volta |
| Tamanho da fila | `xQueueCreate(N, sizeof(item_t))` | Em itens por segundo, quantos segundos de falha a fila absorve antes de começar a descartar; uma fila maior não substitui o fallback, só adia o descarte |
| Tamanho do item | `TAMANHO_MAX` | Maior item esperado, com folga; um item maior é descartado em `xQueueSend`, antes de chegar ao fallback |
| Endpoint | Kconfig (`CONFIG_ENDPOINT_URL`) | Trocar de destino sem recompilar |

### HTTPS

HTTPS vem habilitado por padrão no `esp_http_client` (`CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS`); mudar `http://` para `https://` na URL já ativa o TLS. Falta decidir como validar o certificado do servidor:

| Opção | Campo | Quando usar |
| --- | --- | --- |
| Certificado raiz próprio | `cert_pem` (PEM) | Um servidor específico, certificado conhecido com antecedência |
| Pacote de CAs públicas | `crt_bundle_attach` | Um serviço com certificado de uma autoridade pública (AWS, qualquer provedor comum) |

Com HTTPS, reaproveitar o mesmo `esp_http_client_handle_t` entre vários envios evita refazer o handshake TLS a cada chamada — a documentação chama isso de conexão persistente e recomenda fazer o máximo de requisições possível no mesmo handle. Isso muda a função de envio: em vez de `init` e `cleanup` a cada item, o handle vive durante a task inteira, e a URL de cada requisição é trocada com `esp_http_client_set_url()` quando precisar. Não testado neste guia.

## Problemas comuns

| Sintoma | Causa provável | Ação |
| --- | --- | --- |
| A task trava por vários segundos a cada item, com o destino fora do ar | Disjuntor ausente ou não verificado antes de `enviar()` | Checar `pode_tentar()` antes de toda chamada de envio, inclusive no reenvio |
| Itens somem sem aparecer no fallback | A fila do FreeRTOS encheu antes de `entregar()` rodar | Aumentar a fila, reduzir o timeout do envio, ou aceitar a perda como limite conhecido |
| O reenvio nunca esvazia o arquivo, mesmo com o destino no ar | O `stop`/`parou` da primeira falha ficou antes de testar `pode_tentar()`, ou o disjuntor nunca fecha | Conferir a ordem: primeiro `pode_tentar()`, só então chamar `enviar` |
| Linhas enviadas duas vezes depois de um reset | O reset aconteceu entre a resposta de sucesso e a remoção da linha do fallback | Limite conhecido da troca de arquivo não atômica; se o destino precisar de "exactly once", inclua um identificador único por item para o lado do servidor deduplicar |
| `esp_http_client.h` não encontrado no build | O componente não está no `REQUIRES` | Adicionar `esp_http_client` |
| Erro de certificado só em HTTPS | Nenhum `cert_pem` nem `crt_bundle_attach` configurado | Escolher um dos dois, conforme a tabela de HTTPS |

### Testar a lógica sem hardware

Como a task só depende de três funções (rede disponível, enviar, gravar/reenviar o fallback), dá para testar o disjuntor e a fila num programa de linha de comando no PC, substituindo essas três por versões que simulam rede caindo e voltando, com um relógio controlado pelo teste em vez de `esp_timer_get_time()` real. Foi assim que a lógica deste padrão foi validada antes do primeiro build no hardware.

## Checklist para reaproveitar

- [ ] Escrever as três funções que o padrão precisa (`rede_disponivel`, `enviar`, `gravar_fallback`/`reenviar_fallback`), cada uma testável isoladamente
- [ ] Declarar `esp_http_client` no `REQUIRES`, mais o componente de cada mecanismo de fallback escolhido (SD, NVS, outro)
- [ ] Definir o tamanho do item e da fila a partir da frequência real de itens, não de um valor arbitrário
- [ ] Implementar o disjuntor e confirmar que `pode_tentar()` é checado antes de toda chamada de envio, incluindo dentro do reenvio
- [ ] Fazer o reenvio parar na primeira falha, não tentar o arquivo inteiro a cada ciclo
- [ ] Testar a lógica no PC com funções simuladas, antes do primeiro build no hardware
- [ ] Rodar o teste de três fases no hardware: destino no ar, queda, volta — conferir que nada se perde e que o reenvio respeita a ordem
- [ ] Se migrar para HTTPS, escolher entre `cert_pem` e `crt_bundle_attach`, e avaliar reaproveitar o `esp_http_client_handle_t` entre envios
- [ ] Documentar os limites aceitos (janela de reset na troca de arquivo, duplicatas possíveis, perda em queda mais longa que a fila) em vez de assumir que o padrão é perfeito

### Fontes

Consultadas em 5 de outubro de 2026, por trechos de página.

- [ESP HTTP Client (latest)](https://esp-idf.readthedocs.io/en/latest/api-reference/protocols/esp_http_client.html): `esp_http_client_perform` bloqueia por padrão, reaproveitar o mesmo handle para conexões persistentes, nunca chamar `esp_http_client_perform` duas vezes ao mesmo tempo com o mesmo handle.
- [ESP HTTP Client (v5.3-beta1)](https://docs.espressif.com/projects/esp-idf/en/v5.3-beta1/esp32/api-reference/protocols/esp_http_client.html): HTTPS habilitado por padrão (`CONFIG_ESP_HTTP_CLIENT_ENABLE_HTTPS`), `cert_pem` e `crt_bundle_attach` para validar o servidor, `REQUIRES esp_http_client` no CMakeLists.

O restante (estrutura de fila, disjuntor, reenvio de arquivo) é um padrão de projeto, não uma API específica da Espressif — validado em um projeto real (ESP32, ESP-IDF v6.0.2), não em documentação de terceiros.
