# Monitor de BPM com Alarme — STM32F411

Projeto final de Sistemas Embarcados I. Protótipo didático, sem finalidade diagnóstica.

## Sobre o projeto
O sensor MAX30102 (I2C) lê o batimento pelo dedo e o STM32F411 calcula o BPM.
O valor aparece num display OLED (SPI) e é enviado ao PC pela UART.
Quando o BPM sai da faixa ajustada pelos botões, um LED vermelho acende e o buzzer toca.

## Equipe
| Integrante | Responsável por |
|---|---|
| Nicholas | Hardware (esquemático, layout, protoboard), SPI/OLED e buzzer |
| Henrique Akira | Firmware (GPIO, I2C, EXTI, sensor, algoritmo, integração) |
| Maria Luisa Gonçalves Ferreira | Validação, documentação, UART e scripts Python |

## Estrutura do repositório
- `firmware/` — código do STM32 (drivers, dispositivos, aplicação)
- `hardware/` — esquemático e layout no KiCad, Gerbers, vista 3D, fotos
- `tools/` — scripts Python e de depuração (GDB)
- `docs/` — proposta, capturas de sinal e relatório

## Como compilar e gravar
(a preencher quando o firmware estiver pronto)

## Entrega
14/12/2026

