# Monitor de Frequência Cardíaca (BPM) com Alarme — STM32F411

**Integrantes:** [Integrante A] · [Integrante B] · [Integrante C]
**Entrega:** 14/12/2026 · **Versão 2:** sem fabricação de placa

> **Aviso:** protótipo **didático**, sem finalidade diagnóstica. Os limites de alarme são ilustrativos.

> Valores marcados **(conferir)** devem ser confirmados nos datasheets; preços são **estimativas**.

---

## 1. Objetivo

Construir um **monitor de frequência cardíaca** que:

- lê o sinal do dedo com o sensor óptico **MAX30102** via **I2C**;
- calcula os **batimentos por minuto (BPM)** no firmware;
- mostra o BPM num **display OLED via SPI** e envia os dados ao PC via **UART**;
- acende LED e toca buzzer (**alarme**) quando o BPM sai de uma faixa ajustável por **botões**.

**Hardware:** o protótipo é montado em **protoboard**, com o módulo **Black Pill (STM32F411)**, os módulos do sensor e do display e os componentes discretos. O **esquemático e o layout** da placa são projetados no KiCad e entregues com Gerbers prontos, mas a placa **não é fabricada**.

---

## 2. Equipe

| Integrante | Responsável por | Também faz | Revisa |
|---|---|---|---|
| **[Integrante A]** | **Hardware:** esquemático, layout, montagem na protoboard | **Driver SPI + OLED** (2º barramento) e **PWM do buzzer** (timer) | Drivers |
| **[Integrante B]** | **Firmware:** drivers GPIO/I2C/**EXTI**, sensor, algoritmo de BPM e integração | — | Esquemático e layout |
| **[Integrante C]** | **Validação e documentação:** testes, capturas, README e relatório | Driver UART, gráfico no PC, protótipo do algoritmo em Python | Aplicação |

- Todos têm **commits** no repositório; toda mudança passa por revisão de outro integrante.
- **Escopo de 3 alunos:** o edital pede obrigatórios + (2 opcionais **ou** 2º barramento **ou** fabricação). Sem a fabricação, o grupo cumpre **os dois caminhos restantes**:
  - **2º barramento:** SPI com o display OLED;
  - **2 opcionais:** EXTI (sensor avisa quando há dados) e timer (PWM do buzzer).
- Se um item atrasar, o critério continua atendido pelo outro caminho.

---

## 3. Requisitos obrigatórios

### 3.1 Plataforma

- **MCU:** STM32F411CEU6 no módulo **Black Pill** — Cortex-M4F com FPU, barato e fácil de achar; encaixa direto na protoboard.
- **Gravação e depuração:** **ST-LINK V2** + OpenOCD + GDB via SWD.
- **Compilação:** `arm-none-eabi-gcc` + `Makefile`; headers CMSIS só para nomes de registradores (**sem HAL**).
- **UART para o PC:** adaptador USB-serial CH340.
- **Repositório Git** com README explicando build, gravação, depuração e montagem.

### 3.2 Esquemático

| Folha | Conteúdo principal |
|---|---|
| Alimentação | 5 V da Black Pill → fusível PTC → diodo 1N5819 → TVS → regulador **MCP1700 3,3 V** para os periféricos |
| MCU | Black Pill (2 × 20 pinos) e suas ligações |
| Comunicação | Conector do sensor (I2C + INT) com pull-ups de 2,2 kΩ; conector do OLED (SPI); conector da UART |
| Interface | 2 botões com filtro RC, 2 LEDs, buzzer com transistor BC547 |

- **Circuito mínimo do MCU:** já vem na Black Pill; o relatório explica como o módulo resolve cada item.
- **Consumo (estimativa):** ~28 mA típico, **~87 mA no pior caso** → regulador de 250 mA com folga.
- **Queda de tensão:** 5 V − diodo − PTC ≈ 4,55 V na entrada do regulador; precisa de ≥ 3,5 V → **OK**. Dissipação ≈ 0,09 W.
- **Proteções:** resistores série nas entradas e nos barramentos, pull-ups, LEDs com 270 Ω, diodo no buzzer.
- **ERC** sem erros.
- **O mesmo circuito do esquemático é montado na protoboard**, para que os testes validem o projeto.

### 3.3 Layout (projeto, sem fabricação)

- **2 camadas** com **plano de terra** e vias metalizadas (regras padrão de fabricante, já que não há fabricação local a respeitar).
- Componentes de furo passante, iguais aos usados na protoboard.
- Trilhas de I2C e SPI curtas; buzzer longe dos barramentos; **test points** previstos.
- **DRC** sem erros; entregar arquivos KiCad, PDFs, **Gerbers**, BOM e **vista 3D**.

### 3.4 GPIO

| Pino | Função |
|---|---|
| PA0 / PA1 | Botões MODO e AJUSTE (entrada, ativo em nível baixo) |
| PA8 / PA9 | LED verde (batimento) e LED vermelho (alarme) |
| PB4 | Buzzer — PWM do TIM3 |
| PB1 | INT do sensor — EXTI1 |
| PB8 / PB9 | I2C1 (SCL / SDA) |
| PB13 / PB15 + PB12, PB14, PA4 | SPI2 (SCK / MOSI) + CS, DC, RES do OLED |
| PA2 / PA3 | USART2 (TX / RX) |

- Configuração **direto nos registradores** (`MODER`, `PUPDR`, `IDR`, `BSRR`).
- **Debounce:** filtro RC de 1 ms + confirmação de 20 ms no software (bounce típico de 1–10 ms; será medido).

### 3.5 Barramento I2C (obrigatório) e SPI (2º barramento)

| | I2C — sensor MAX30102 | SPI — display OLED |
|---|---|---|
| Endereço / seleção | `0x57` (escrita `0xAE`, leitura `0xAF`) | Pino CS |
| Velocidade | ~381 kHz (Fast Mode) | 4 MHz, modo 0 |
| Dados | Leitura da FIFO: 3 bytes por amostra, 100 amostras/s | 1024 bytes por tela, 10 vezes por segundo |
| Hardware | Pull-ups de **2,2 kΩ** (calculado entre 0,97 e 7,1 kΩ) | Resistores série de 33 Ω |

- **Validação:** capturas no **analisador lógico** (leitura do PART_ID = `0x15`, leitura da FIFO, inicialização do OLED) e tempo de subida do I2C no osciloscópio.
- **Protoboard:** fios curtos nos barramentos; a capacitância extra da protoboard é considerada no tempo de subida do I2C (se passar do limite, baixar para 100 kHz).
- **Ponto de atenção:** o I2C do F411 exige uma sequência especial nos últimos bytes de leitura (RM0383) — é o primeiro driver a ser feito.

### 3.6 Itens adicionais: EXTI e timer

- **EXTI1 (PB1):** o MAX30102 baixa o pino INT quando a FIFO tem dados; a interrupção sinaliza a leitura, sem ficar consultando o sensor em laço.
- **TIM3 (PB4):** PWM de ~2–4 kHz **(conferir a frequência do buzzer)** no canal 1; o duty cycle liga e desliga o alarme sem ocupar a CPU.

### 3.7 Integração e depuração

- **Fluxo:** sensor (I2C + EXTI) → algoritmo → display (SPI) + LEDs (GPIO) + buzzer (TIM3) + PC (UART).
- **Algoritmo:** remove a parte constante do sinal, filtra, detecta picos e calcula BPM = 60 000 / intervalo médio entre batimentos. Testado antes em Python.
- **Firmware em camadas:** drivers → dispositivos (sensor, display) → aplicação.
- **GDB ao vivo:** breakpoint a cada batimento, leitura da variável `bpm_atual` e dos registradores `I2C1->SR1`, `GPIOA->IDR`, `TIM3->CCR1`, `EXTI->PR`.

### 3.8 Validação

| Teste | Critério de sucesso |
|---|---|
| T0 – Montagem | Ligações da protoboard conferidas com o esquemático; sem curtos |
| T1 – Alimentação | 3,3 V ± 3% |
| T2/T3 – I2C | Sensor responde em `0x57`; PART_ID = `0x15`; tempo de subida < 300 ns |
| T4 – SPI | Display mostra a tela; captura da inicialização |
| T5 – Botões | 50 apertos = 50 eventos |
| T6 – EXTI / amostragem | 1 interrupção por bloco de dados; 100 ± 2 amostras/s |
| T7 – **Precisão** | Erro ≤ ±5 BPM em 90% das leituras (comparado com oxímetro/relógio) |
| T8/T9 – Lógica | Sem dedo → mensagem em 2 s; fora da faixa → alarme em 3 s |
| T10 – Timer | Frequência do PWM medida no osciloscópio dentro de ±2% |
| T11/T12 – Consumo e UART | ≤ 100 mA; dados no PC sem erro |

**Relatório:** esquemático, layout (2D e 3D), fotos da montagem, capturas, tabela esperado × obtido, conclusões e parágrafo de contribuição de cada integrante.

---

## 4. Itens adicionais

- ❌ ~~Fabricação e montagem da placa~~ — **não será feita**
- ✅ **Segundo barramento** (SPI com o OLED)
- ✅ **EXTI** (sensor avisa quando há dados)
- ✅ **Timer** (PWM do buzzer)

---

## 5. Proposta

### Diagrama de blocos

```
                    PC (USB) ──► USB-C da Black Pill
                                       │ 5V
 ┌─────────────────────────────────── PROTOBOARD ─────────────────────────────────────┐
 │                                     ▼                                              │
 │  [PTC]─[1N5819]─[TVS] ──► 5V_P ──► [LDO MCP1700 3,3V] ──► 3V3_P (periféricos)      │
 │                                                                                    │
 │  [MAX30102]◄─ I2C1 ~381 kHz ─┐                  ┌─ SPI2 4 MHz ─►[OLED SSD1306]     │
 │    INT ─────── EXTI1 ───────►│  ┌────────────┐  │                                  │
 │  [pull-ups 2,2k + 33 Ω]      └─►│ BLACK PILL │◄─┘                                  │
 │                                 │ STM32F411  │──► LED verde (batimento)            │
 │  [BTN MODO  ]──RC──────────────►│            │──► LED vermelho (alarme)            │
 │  [BTN AJUSTE]──RC──────────────►│            │──► Buzzer (BC547, PWM TIM3)         │
 │                                 │            │──► UART ──► CH340 ──► PC            │
 │                                 └─────┬──────┘                                     │
 └───────────────────────────────────────┼────────────────────────────────────────────┘
                                         │ SWD
                                     ST-LINK V2 ──► PC (GDB / OpenOCD)
```

### Componentes principais

| Componente | Qtd | Referência |
|---|---|---|
| Black Pill STM32F411CEU6 | 1 | https://github.com/WeActStudio/WeActStudio.MiniSTM32F4x1 |
| ST-LINK V2 + adaptador USB-serial CH340 | 1 + 1 | — |
| Módulo MAX30102 | 1 | https://www.analog.com/en/products/max30102.html |
| Display OLED 0,96" SSD1306 (SPI) | 1 | Datasheet SSD1306 do fornecedor |
| Regulador MCP1700-3,3 V (TO-92) | 1 | https://www.microchip.com/en-us/product/MCP1700 |
| PTC 250 mA, 1N5819, P6KE6.8A, BC547, 1N4148 | 1 cada | — |
| Buzzer, 3 LEDs, 2 botões, ~15 resistores, ~8 capacitores | — | — |
| Protoboard 830 pontos + jumpers | 1 + kit | — |

### Cronograma

**Marcos:** proposta revisada **17/11** · esquemático **21/11** · protótipo montado **23/11** · layout **28/11** · congelamento do código **08/12** · testes finais **10/12** · relatório **11/12** · ensaio **12–13/12** · **entrega 14/12**.

| Período | A – Hardware | B – Firmware | C – Validação / Documentação |
|---|---|---|---|
| **16–18/11** | Comprar componentes; começar esquemático | Toolchain, Makefile, LED piscando; GDB | **Proposta revisada (17/11)**; criar repositório |
| **19–23/11** | **Esquemático + ERC (21/11)**; **monta na protoboard (23/11)** | GPIO + debounce; **driver I2C** + PART_ID | Driver UART + gráfico no PC; plano de testes; testa T0, T1 |
| **24–28/11** | **Layout + DRC (28/11)**; driver SPI/OLED | Driver do sensor + **EXTI** | Captura I2C; algoritmo em Python; revisa esquemático |
| **29/11–07/12** | **PWM do buzzer (TIM3)**; telas do OLED | Algoritmo em C; **integração completa** | Testes T2–T12 com capturas; README |
| **08–10/12** | Correções; texto de hardware | **Congela o código (08/12)**; roteiro do GDB | **Testes finais (10/12)** |
| **11–14/12** | Revisão + ensaio | Revisão + ensaio | **Relatório (11/12)**; checklist; **entrega** |

Sem a fabricação, o firmware pode ser testado no hardware real desde **23/11**, o que tira o maior risco do cronograma anterior.

### Custo (estimativa)

| Item | Faixa |
|---|---|
| Black Pill | R$ 35–60 |
| ST-LINK V2 + CH340 | R$ 35–65 |
| MAX30102 + OLED | R$ 35–65 |
| Componentes discretos | R$ 20–40 |
| Protoboard + jumpers | R$ 15–35 |
| **Total** | **≈ R$ 140–265** (≈ R$ 47–88 por pessoa) |

Analisador lógico, osciloscópio e fonte: **emprestados do laboratório**.

---

## 6. Restrições

- **Prazo:** firmware feito em paralelo desde o primeiro dia; protótipo na protoboard na primeira semana.
- **Componentes acessíveis:** todos de loja local.
- **CAD:** KiCad (gratuito).
- **Sem HAL/CubeMX:** todos os drivers escritos direto nos registradores.

---

## 7. Avaliação

| Critério (peso) | O que o grupo vai mostrar |
|---|---|
| Proposta (10%) | Este documento e o cronograma cumprido |
| Esquemático (20%) | Alimentação e proteções calculadas, ERC limpo |
| Layout (20%) | 2 camadas com plano de terra, DRC limpo, Gerbers e vista 3D (sem fabricação) |
| Firmware (20%) | Drivers por registrador, SPI + EXTI + timer, camadas separadas, integração completa |
| Validação (15%) | Testes T0–T12 com capturas e tabela esperado × obtido |
| Defesa (15%) | GDB ao vivo; todos sabem explicar todas as partes |

**Pendências antes de começar:**

- [ ] Professor confirmar que o layout pode ser entregue sem fabricação
- [ ] Professor confirmar que SPI + EXTI + timer cumprem o escopo de 3 alunos
- [ ] Preencher os nomes dos integrantes
