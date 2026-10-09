# ESP32 — Tutoriais

Registro didático do aprendizado prático de ESP32 com ESP-IDF, organizado em tutoriais sequenciais.

**Hardware em uso:** ESP32 (módulo DevKit)
**Ambiente:** ESP-IDF (`idf.py`) + VS Code

## Tutoriais

| # | Tutorial | Categoria | Conteúdo |
| --- | --- | --- | --- |
| 01 | [UART](01-uart/README.md) | Periférico | Driver UART do ESP-IDF (fila de eventos), recepção de linhas JSON do STM32, troubleshooting de ruído de linha |
| 02 | [Wi-Fi manager em modo estação](02-wifi-manager/README.md) | Conectividade | Conexão Wi-Fi STA com credenciais no `menuconfig`, reconexão automática e módulo `wifi_manager` reaproveitável |
| 03 | [Cartão SD por SPI](03-sd-spi/README.md) | Armazenamento | Montagem de cartão SD por SPI como diretório FAT (`/sdcard`), leitura e gravação de arquivos |
| 04 | [Envio HTTP com fallback local](04-http-fallback/README.md) | Integração | Padrão "envia agora; se não der, guarda e reenvia": fila, task de envio e disjuntor para servidor fora do ar. Depende de 02 e 03 |
| 05 | [Wi-Fi em modo duplo (STA + AP de configuração)](05-wifi-ap-sta/README.md) | Conectividade | AP de configuração com fallback automático após N rodadas falhando, troca AP→STA sem reboot, persistência de credenciais em NVS. Depende de 02 |
| 06 | [Servidor HTTP embarcado (painel web + API JSON)](06-http-server/README.md) | Integração | `esp_http_server` com arquivos estáticos via `EMBED_TXTFILES` e endpoints JSON; dois problemas reais de símbolo/terminador nulo documentados |
| 07 | [Key/value genérico em NVS](07-nvs-settings-store/README.md) | Armazenamento | Configurações string/inteiro com valor padrão garantido, para conjuntos de chaves que crescem com o tempo |
