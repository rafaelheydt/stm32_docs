# ESP-IDF: key/value genérico em NVS — guia de referência

## Resumo

Este guia monta um armazenamento genérico de configurações em NVS — string e inteiro, cada um por uma chave de texto — para configurações cujo conjunto cresce com o tempo (por exemplo, expostas numa página web) sem precisar escrever uma função nova para cada campo.

| Item | Valor validado |
| --- | --- |
| Chip | ESP32 clássico (módulo WROOM-32) |
| ESP-IDF | v6.0.2 |
| Armazenamento | NVS, namespace próprio (`"settings"`) |
| Tipos suportados | string (`nvs_get_str`/`nvs_set_str`) e inteiro 32 bits (`nvs_get_i32`/`nvs_set_i32`) |
| Comportamento em erro | Sempre devolve um valor usável — o valor padrão informado pelo chamador |

Use este padrão quando as chaves variam ou crescem (um formulário web que ganha campos novos, configurações opcionais). Quando o conjunto de campos é fixo e conhecido de antemão (duas ou três credenciais, por exemplo), prefira campos dedicados — ver o módulo `wifi_creds` no [guia 05](../05-wifi-ap-sta/README.md), que persiste SSID/senha como campos tipados em vez de key/value genérico.

## Pré-requisitos

### CMake

```cmake
idf_component_register(SRCS "main.c" "settings_store.c"
                    INCLUDE_DIRS "."
                    REQUIRES nvs_flash)
```

### NVS inicializada

`nvs_flash_init()` precisa já ter rodado (normalmente no `app_main`, como em qualquer uso de NVS — ver o guia 02 para o tratamento de `ESP_ERR_NVS_NO_FREE_PAGES`/`ESP_ERR_NVS_NEW_VERSION_FOUND`).

## O limite de 15 caracteres por chave

A NVS limita o nome de cada chave a **15 caracteres** — não é uma escolha deste módulo, é um limite real da biblioteca. Uma chave maior falha em tempo de execução (`nvs_set_str`/`nvs_set_i32` retornam erro), não em tempo de compilação, então o erro só aparece quando alguém tenta salvar com aquela chave específica. Validar o tamanho antes de cada escrita evita que esse erro apareça silenciosamente em produção:

```c
bool settings_set_string(const char *key, const char *value)
{
    if (key == NULL || value == NULL || strlen(key) > 15) {
        return false;
    }
    // ...
}
```

## Código mínimo

### Leitura com valor padrão garantido

```c
#define SETTINGS_VALUE_MAX_LEN 128

void settings_get_string(const char *key, char *out, size_t out_len, const char *default_value)
{
    nvs_handle_t handle;
    if (nvs_open("settings", NVS_READONLY, &handle) != ESP_OK) {
        strncpy(out, default_value ? default_value : "", out_len - 1);
        out[out_len - 1] = '\0';
        return;
    }

    size_t len = out_len;
    esp_err_t err = nvs_get_str(handle, key, out, &len);
    nvs_close(handle);

    if (err != ESP_OK) {
        // ESP_ERR_NVS_NOT_FOUND: chave nunca foi salva — caso normal, não um erro
        // ESP_ERR_NVS_INVALID_LENGTH: buffer "out" pequeno demais pro valor salvo
        strncpy(out, default_value ? default_value : "", out_len - 1);
        out[out_len - 1] = '\0';
    }
}
```

**Por que a função nunca "falha" para quem chama:** em qualquer cenário de erro — namespace nunca criado, chave nunca salva, buffer pequeno demais — a função devolve `default_value` em vez de deixar `out` com conteúdo indefinido. Quem chama não precisa checar um código de retorno antes de usar o valor.

### Gravação com commit explícito

```c
bool settings_set_string(const char *key, const char *value)
{
    if (key == NULL || value == NULL || strlen(key) > 15) {
        return false;
    }

    nvs_handle_t handle;
    if (nvs_open("settings", NVS_READWRITE, &handle) != ESP_OK) {
        return false;
    }

    esp_err_t err = nvs_set_str(handle, key, value);
    if (err == ESP_OK) {
        err = nvs_commit(handle);   // sem isto, a escrita pode não persistir
    }
    nvs_close(handle);

    return err == ESP_OK;
}
```

`nvs_set_str`/`nvs_set_i32` escrevem na página de NVS, mas `nvs_commit()` é o que garante a gravação — sem ele, dependendo do momento de um reset, a alteração pode não sobreviver ao reboot.

### Inteiro — mesmo padrão, outro par de funções

```c
int settings_get_int(const char *key, int default_value)
{
    nvs_handle_t handle;
    if (nvs_open("settings", NVS_READONLY, &handle) != ESP_OK) {
        return default_value;
    }

    int32_t value = default_value;
    esp_err_t err = nvs_get_i32(handle, key, &value);
    nvs_close(handle);

    return (err == ESP_OK) ? (int)value : default_value;
}
```

## Uso típico (endpoint HTTP editável)

```c
char endpoint[SETTINGS_VALUE_MAX_LEN + 1];
settings_get_string("http_endpoint", endpoint, sizeof(endpoint), CONFIG_HTTP_ENDPOINT_URL);
```

O chamador nunca precisa saber se o valor veio da NVS ou do padrão do Kconfig — a função resolve isso internamente. É esse desacoplamento que permite ao [servidor HTTP (guia 06)](../06-http-server/README.md) expor um endpoint `/api/settings` genérico: a tabela de configurações no `web_ui.c` só lista `{chave, rótulo, valor padrão}` e delega a leitura/escrita real para este módulo.

## Problemas comuns

| Sintoma | Causa provável | Ação |
| --- | --- | --- |
| `settings_set_string` retorna `false` sem log claro do motivo | Chave com mais de 15 caracteres | Validar `strlen(key) <= 15` antes de chamar, ou escolher nomes de chave mais curtos desde o início |
| Valor salvo "some" depois de um reset | `nvs_commit()` não foi chamado, ou falhou silenciosamente (retorno ignorado) | Sempre checar o retorno de `nvs_commit` e só considerar a gravação bem-sucedida se ele for `ESP_OK` |
| Primeira leitura de uma chave nova devolve o valor padrão, e isso é tratado como bug | Comportamento esperado: `ESP_ERR_NVS_NOT_FOUND` antes da primeira gravação não é um erro de verdade | Nenhuma ação — é o comportamento desejado do padrão |
| `nvs_get_str` retorna `ESP_ERR_NVS_INVALID_LENGTH` | Buffer de saída menor que o valor salvo | Aumentar o buffer (`SETTINGS_VALUE_MAX_LEN`) para cobrir o maior valor esperado, com folga |
| Dois módulos pisando nos dados um do outro | Namespace (`nvs_open`) compartilhado entre funcionalidades sem relação | Dar um namespace próprio a cada "família" de dados (como este projeto faz: `"settings"` aqui, `"wifi_creds"` no guia 05) |

## Checklist para reaproveitar

- [ ] Escolher um namespace próprio (não reaproveitar o de outro módulo)
- [ ] Validar o tamanho da chave (≤ 15 caracteres) antes de toda escrita
- [ ] Sempre chamar `nvs_commit()` depois de `nvs_set_*`, e checar o retorno
- [ ] Tratar `ESP_ERR_NVS_NOT_FOUND` como caso normal (primeira leitura), não como erro a logar
- [ ] Dimensionar o buffer de leitura (string) para o maior valor esperado, com folga
- [ ] Decidir, por campo, se ele merece um nome de função dedicado (poucos campos fixos, como `wifi_creds`) ou se cabe no key/value genérico (muitos campos ou crescendo)

### Fontes

- [NVS API do ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/nvs_flash.html): limite de 15 caracteres por chave, necessidade de `nvs_commit`, e os códigos de erro (`ESP_ERR_NVS_NOT_FOUND`, `ESP_ERR_NVS_INVALID_LENGTH`).
- O padrão de "nunca falha para quem chama, sempre devolve um valor usável" é o desenho deste projeto, validado em hardware real, não uma recomendação explícita da documentação oficial.
