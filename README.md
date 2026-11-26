# AirSense — Monitor de qualidade do ar para ambientes internos

Sistema embarcado com **ESP32** que mede temperatura, umidade, CO₂, poeira e pressão, classifica a qualidade do ar em três níveis (**OK / ATENÇÃO / ALERTA**) com base na norma da ANVISA, sinaliza localmente (LEDs, buzzer e display OLED) e envia as leituras para uma API pelo Wi-Fi.

> Projeto extensionista da disciplina de Microcontroladores — Estácio Ribeirânia — **Grupo 12**.


---

## Sumário

- [Funcionalidades](#funcionalidades)
- [Como o ar é classificado](#como-o-ar-é-classificado)
- [Hardware](#hardware)
- [Esquema de ligação](#esquema-de-ligação)
- [Estrutura do projeto](#estrutura-do-projeto)
- [Como executar](#como-executar)
- [Configuração](#configuração)
- [Integração com a API](#integração-com-a-api)
- [Arquitetura do firmware](#arquitetura-do-firmware)
- [Limitações e próximos passos](#limitações-e-próximos-passos)
- [Equipe](#equipe)

---

## Funcionalidades

- Leitura a cada **1 s** de temperatura, umidade, CO₂, poeira (material particulado) e pressão atmosférica.
- Classificação automática em **OK**, **ATENÇÃO** e **ALERTA**.
- **LEDs** verde / amarelo / vermelho indicando o nível atual.
- **Buzzer** (PWM por hardware, periférico LEDC) que apita em ALERTA.
- **Botão de silenciar**: pausa o buzzer por 15 minutos; um segundo aperto cancela o mudo.
- **Display OLED 128×64** (I²C) com todas as leituras, o nível, o estado do Wi-Fi (`W`) e do mudo (`M`).
- **Monitor serial** (115200 baud) com uma linha por leitura.
- **Envio para API** via HTTP/HTTPS (POST em JSON) a cada 5 s, com reconexão automática ao Wi-Fi.
- Tolerância a falhas: se o DHT22 falhar, temperatura e umidade saem da classificação (sem alerta falso) e vão como `null` no JSON.
- Funciona **sem Wi-Fi e sem API**: se `secrets.h` não existir, o dispositivo opera normalmente, apenas sem enviar dados.

## Como o ar é classificado

Limites baseados na **ANVISA, Resolução RE nº 9/2003** (qualidade do ar interior em ambientes climatizados de uso coletivo).

| Grandeza | ATENÇÃO (80% do limite) | ALERTA (limite da norma) |
|---|---|---|
| CO₂ | > 800 ppm | > 1000 ppm |
| Poeira (aerodispersóides) | > 64 µg/m³ | > 80 µg/m³ |

| Grandeza | Faixa de conforto (verão) |
|---|---|
| Temperatura | 23 – 26 °C |
| Umidade relativa | 40 – 65 % |

**Regras (em ordem de prioridade):**

1. **ALERTA** — CO₂ ou poeira acima do limite da norma.
2. **ATENÇÃO** — CO₂ ou poeira acima de 80% do limite.
3. **ATENÇÃO** — estando tudo OK, temperatura ou umidade fora da faixa de conforto.
4. **OK** — nenhuma das anteriores.

A pressão atmosférica é medida e enviada à API, mas **não** entra na classificação.

## Hardware

| Componente | Função | Pino (ESP32) |
|---|---|---|
| ESP32 DevKit C v4 | Microcontrolador (Wi-Fi integrado, ADC, LEDC, I²C) | — |
| DHT22 | Temperatura e umidade | GPIO 15 |
| Potenciômetro (simula SCD40) | CO₂ | GPIO 34 (ADC1) |
| Potenciômetro (simula GP2Y1010) | Poeira | GPIO 35 (ADC1) |
| Potenciômetro (simula BMP280) | Pressão | GPIO 32 (ADC1) |
| LED verde + resistor 220 Ω | Nível OK | GPIO 25 |
| LED amarelo + resistor 220 Ω | Nível ATENÇÃO | GPIO 26 |
| LED vermelho + resistor 220 Ω | Nível ALERTA | GPIO 27 |
| Buzzer | Alarme sonoro (PWM, 2 kHz) | GPIO 33 |
| Botão (pull-up interno) | Silenciar alarme | GPIO 18 |
| OLED SSD1306 128×64 (0x3C) | Display, via I²C | SDA 21 / SCL 22 |

> Todos os sinais analógicos ficam no **ADC1**, porque o ADC2 não funciona com o Wi-Fi ligado.

**Periféricos e recursos do microcontrolador utilizados:** GPIO, ADC de 12 bits, PWM (LEDC), I²C, UART, interrupção externa (botão), Wi-Fi e `millis()` para temporização.

### Simulação × hardware real

Em `MODO_SIMULACAO = 1`, os potenciômetros fazem o papel dos sensores reais, mapeados para as faixas típicas de cada um:

| Sinal | Sensor real previsto | Faixa simulada |
|---|---|---|
| CO₂ | SCD40 (I²C) | 400 – 2000 ppm |
| Poeira | GP2Y1010 (ADC) | 0 – 150 µg/m³ |
| Pressão | BMP280 (I²C) | 950 – 1050 hPa |

O DHT22 é real no Wokwi. A leitura dos sensores reais (SCD40, GP2Y1010, BMP280) **ainda não está implementada**; com `MODO_SIMULACAO 0` a compilação para com um `#error` explicativo.

## Esquema de ligação

O circuito completo está em [`diagram.json`](diagram.json) (formato Wokwi). Resumo:

```
ESP32 3V3  ── OLED VCC, DHT22 VCC, Pots VCC
ESP32 GND  ── OLED GND, DHT22 GND, Pots GND, cátodos dos LEDs, buzzer, botão
GPIO 21/22 ── OLED SDA/SCL
GPIO 15    ── DHT22 SDA
GPIO 34/35/32 ── Pot CO₂ / Pot poeira / Pot pressão
GPIO 25/26/27 ── resistor 220 Ω ── LED verde / amarelo / vermelho
GPIO 33    ── Buzzer
GPIO 18    ── Botão ── GND
```

## Estrutura do projeto

```
.
├── sketch.ino        # setup(), loop() e desenho do display
├── config.h          # modo de operação, pinos, limiares, tempos e rede
├── sensores.h        # leitura dos sensores (DHT22 + potenciômetros simulados)
├── alertas.h         # classificação, LEDs, buzzer (PWM) e botão de mudo
├── rede.h            # Wi-Fi e envio das leituras para a API
├── secrets.example.h # modelo de credenciais (copiar para secrets.h)
├── diagram.json      # circuito do Wokwi
├── libraries.txt     # bibliotecas do Wokwi
└── README.md
```

## Como executar

### Opção 1 — Simulador Wokwi (recomendado)

1. Abra o projeto: https://wokwi.com/projects/480982867950051329
2. Clique em **▶ Start Simulation**.
3. Gire os potenciômetros e altere o DHT22 para ver os níveis mudarem. Clique no botão azul para silenciar o buzzer.
4. (Opcional) Para enviar dados à API, crie uma aba `secrets.h` — veja [Configuração](#configuração).

### Opção 2 — VS Code + PlatformIO / Arduino IDE

Bibliotecas necessárias:

- Adafruit GFX Library
- Adafruit SSD1306
- DHT sensor library

O firmware é compatível com o core do ESP32 versões **2.x e 3.x** (a API do LEDC é tratada nos dois casos).

## Configuração

Todos os parâmetros ajustáveis ficam em [`config.h`](config.h), sem necessidade de mexer na lógica:

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `MODO_SIMULACAO` | `1` | `1` = potenciômetros; `0` = sensores reais (não implementado) |
| `INTERVALO_LEITURA_MS` | 1000 | Período de leitura e atualização da tela |
| `INTERVALO_ENVIO_MS` | 5000 | Período de envio para a API |
| `DURACAO_BIPE_MS` / `FREQ_BIPE_HZ` | 100 / 2000 | Duração e frequência do bipe |
| `TEMPO_MUDO_MS` | 15 min | Duração do silenciamento (use 15 s para testar) |
| `TIMEOUT_WIFI_MS` | 15000 | Espera máxima ao conectar |
| `INTERVALO_RECONEXAO_MS` | 30000 | Intervalo entre tentativas de reconexão |
| `TIMEOUT_HTTP_MS` | 5000 | Timeout das requisições |

### Credenciais (`secrets.h`)

Rede e API são configuradas em um arquivo **fora do Git**:

1. Copie `secrets.example.h` para `secrets.h` (no Wokwi, crie uma aba chamada `secrets.h`).
2. Preencha os valores:

```cpp
#define WIFI_SSID   "Wokwi-GUEST"                          // rede virtual do Wokwi, sem senha
#define WIFI_SENHA  ""
#define API_URL     "https://SEU-TUNEL.trycloudflare.com/leituras"
#define API_KEY     "SUA_CHAVE_AQUI"                       // mesma de API_KEY em api/.env
#define DEVICE_ID   "airsense-01"
```

> ⚠️ **Nunca** faça commit do `secrets.h` nem coloque chaves reais no `secrets.example.h`. Garanta que `secrets.h` está no `.gitignore`.

`localhost` não funciona no wokwi.com: exponha a API com um túnel (ex.: Cloudflare Tunnel). No Wokwi do VS Code, é possível usar `http://host.wokwi.internal:8000/leituras`.

## Integração com a API

A cada 5 s o firmware envia um `POST` para `API_URL`, com os cabeçalhos `Content-Type: application/json` e `x-api-key: <API_KEY>`.

**Exemplo de corpo:**

```json
{
  "device": "airsense-01",
  "temperatura": 24.5,
  "umidade": 55.0,
  "co2": 812,
  "poeira": 41.3,
  "pressao": 1012.4,
  "nivel": 1
}
```

- `nivel`: `0` = OK, `1` = ATENÇÃO, `2` = ALERTA.
- Se o DHT22 falhar, `temperatura` e `umidade` são enviados como `null`.
- Respostas: `201` gravado · `401` chave incorreta · `422` dados fora do formato.
- URLs `https://` usam `setInsecure()` (aceita qualquer certificado) — adequado para protótipo; **em produção, valide o certificado** com `setCACert(...)`.

O contrato completo da API está descrito em `api/README.md`.

## Arquitetura do firmware

O `loop()` **não usa `delay()`**: cada tarefa roda em seu próprio intervalo, medido com `millis()`, para que buzzer, tela e envio não travem uns aos outros.

```
loop()
 ├─ atualizarMudo()      → trata o pedido do botão e o fim dos 15 min
 ├─ Tarefa 1 (1 s)       → lerSensores → classificar → sinalizar → tela + serial
 ├─ Tarefa 2 (contínua)  → atualizarBuzzer(): encerra o bipe após 100 ms
 └─ Tarefa 3 (5 s)       → enviarParaAPI() (aguarda o bipe terminar)
```

Decisões de projeto:

- **PWM por LEDC:** a onda do buzzer é gerada em hardware; a CPU fica livre durante o bipe.
- **Interrupção mínima:** a ISR do botão (com debounce de 250 ms) só levanta uma flag; a lógica do mudo roda no `loop()`.
- **Falha do DHT22 ≠ alerta falso:** o `NaN` é preservado e marcado com `dhtOk = false`, em vez de ser trocado por 0.
- **Envio não bloqueante para o alarme:** o POST só ocorre quando o buzzer não está tocando, pois pode levar alguns segundos.
- **Reconexão espaçada:** como reconectar bloqueia o loop por até 15 s, a nova tentativa só acontece a cada 30 s.

## Limitações e próximos passos

- [ ] Implementar a leitura dos sensores reais: **SCD40** (CO₂, I²C), **BMP280** (pressão, I²C) e **GP2Y1010** (poeira, ADC).
- [ ] Validar o certificado do servidor no HTTPS (remover `setInsecure()`).
- [ ] Montagem física do protótipo e testes em campo.
- [ ] Ajustar os limiares conforme o ambiente-alvo (os atuais seguem a RE 9/2003 para ambientes climatizados).

