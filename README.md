# CPU-8Bits

# CPU 8 bits — Arquitetura Simplificada

Projeto de uma CPU de arquitetura simplificada de 8 bits, desenvolvido no simulador [Digital](https://github.com/hneemann/Digital). A CPU implementa uma ALU completa, um circuito de controle com ciclos de busca e execução, e uma memória EEPROM para armazenar instruções e operandos.

---

## Sumário

- [Visão geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [ALU — Unidade Lógica e Aritmética](#alu--unidade-lógica-e-aritmética)
- [Conjunto de instruções](#conjunto-de-instruções)
- [Ciclo de busca e execução](#ciclo-de-busca-e-execução)
- [Unidade de Controle](#unidade-de-controle)
- [Como simular](#como-simular)
- [Exemplos de programas](#exemplos-de-programas)
- [Estrutura do repositório](#estrutura-do-repositório)

---

## Visão geral

A CPU segue uma arquitetura de **endereçamento por pilha (stack addressing)**, onde instrução e operando ficam em endereços consecutivos na memória. Isso simplifica o ciclo de busca: a cada dois pulsos de clock, a CPU busca uma instrução e depois busca o operando, executa a operação e grava o resultado no acumulador.

```
Clock 1 (Fetch):   PC → EEPROM → IR (grava instrução) → PC+1
Clock 2 (Execute): PC → EEPROM → ALU (operando N) → AC/MQ → PC+1
```

---

## Arquitetura

```
         ┌─────────────────────────────────────────────┐
         │           Unidade de Controle               │
         │   FF-D: alterna Q entre 0 (fetch) e 1 (exec)│
         │   Q̄ → en_IR       Q → En_AC                │
         └──────────┬──────────────┬───────────────────┘
                    │              │
         ┌──────────▼──┐      ┌────▼──────────────────────┐
         │     PC      │      │        ALU-completa        │
         │ Reg 8 bits  │      │  Seletor, N, C_AC, En_AC  │
         │ + Somador+1 │      │  C_MQ, En_MQ              │
         └──────┬───────┘      │  Saídas: AC, MQ, Resto   │
                │              └───────────────────────────┘
         ┌──────▼───────┐
         │    EEPROM    │
         │ 8 bits dado  │
         │ 8 bits end.  │
         │ CS=1 OE=1    │
         │ WE=0         │
         └──────┬───────┘
                │ D[7:0]
         ┌──────▼──────────────────┐
         │  IR (Instruction Reg)   │  → Splitter → bits[7:5] → Seletor ALU
         │  Reg 8 bits             │
         │  en = Q̄ (só no fetch)  │
         └─────────────────────────┘
```

### Componentes principais

| Componente | Função |
|---|---|
| **PC** | Registrador de 8 bits + somador com constante 1. Aponta o endereço atual na EEPROM e incrementa a cada clock. |
| **EEPROM** | Memória de instruções e operandos. 8 bits de dado, 8 bits de endereço (256 posições). |
| **IR** | Instruction Register. Grava a instrução lida da EEPROM apenas no ciclo de busca. |
| **Splitter** | Separa os 3 bits mais significativos do IR (opcode) para enviar ao Seletor da ALU. |
| **UC** | Flip-flop D com realimentação Q̄→D. Gera os sinais en_IR e En_AC. |
| **ALU-completa** | Subcircuito com todas as operações, registrador AC e registrador MQ internos. |
| **AC** | Acumulador de 8 bits. Entrada A de todas as operações. Grava resultado no ciclo de execução. |
| **MQ** | Registrador de 8 bits para multiplicação (8 MSB) e divisão (quociente). Só grava em MUL e DIV. |

---

## ALU — Unidade Lógica e Aritmética

A ALU possui 8 operações:

| Opcode (bits 7-5) | Operação | Resultado |
|---|---|---|
| `000` | **ADD** | AC + N → AC |
| `001` | **SUB** | AC − N → AC |
| `010` | **MUL** | AC × N → AC (8 LSB) e MQ (8 MSB) |
| `011` | **DIV** | AC ÷ N → MQ (quociente) e AC (resto) |
| `100` | **SHL** | AC << 1 → AC (shift lógico esquerda) |
| `101` | **SHR** | AC >> 1 → AC (shift lógico direita) |
| `110` | **NAND** | AC NAND N → AC |
| `111` | **XOR** | AC XOR N → AC |

### Registradores

- **AC (Acumulador):** entrada A de todas as operações. Atualizado a cada ciclo de execução.
- **MQ:** atualizado apenas nas operações MUL e DIV. A lógica de habilitação é `En_MQ = Q AND NOT(Seletor[2]) AND Seletor[1]`, garantindo que só muda quando o opcode é `010` ou `011`.

### Shift lógico

| Direção | Bit descartado | Bit inserido |
|---|---|---|
| Esquerda (SHL) | MSB | 0 no LSB |
| Direita (SHR) | LSB | 0 no MSB |

### Multiplicação e divisão

```
MUL:  AC × N = resultado 16 bits
      AC  ← resultado[7:0]   (8 LSB)
      MQ  ← resultado[15:8]  (8 MSB)

DIV:  AC ÷ N
      MQ  ← quociente
      AC  ← resto
```

---

## Conjunto de instruções

A palavra de instrução tem 8 bits. Os **3 bits mais significativos** definem o opcode. Os 5 bits restantes são ignorados.

```
 7   6   5   4   3   2   1   0
┌───┬───┬───┬───┬───┬───┬───┬───┐
│   opcode  │     ignorado      │
└───┴───┴───┴───┴───┴───┴───┴───┘
```

### Bytes de instrução

| Instrução | Byte (hex) |
|---|---|
| ADD | `0x00` |
| SUB | `0x20` |
| MUL | `0x40` |
| DIV | `0x60` |
| SHL | `0x80` |
| SHR | `0xA0` |
| NAND | `0xC0` |
| XOR | `0xE0` |

---

## Ciclo de busca e execução

A CPU usa **endereçamento por operandos sequenciais**: instrução e operando ficam em endereços consecutivos, como uma pilha.

```
Endereço  Conteúdo
  0x00    instrução 1    ← PC aponta aqui no clock 1 (fetch)
  0x01    operando N1    ← PC aponta aqui no clock 2 (execute)
  0x02    instrução 2    ← clock 3 (fetch)
  0x03    operando N2    ← clock 4 (execute)
  ...
```

### Sinais de controle por estado

| Estado | Q | en_IR | En_AC | O que acontece |
|---|---|---|---|---|
| Fetch | 0 | 1 | 0 | IR grava instrução da EEPROM. PC+1. |
| Execute | 1 | 0 | 1 | ALU recebe N da EEPROM, executa, AC grava. PC+1. |

---

## Unidade de Controle

Implementada com um único **Flip-flop D** cuja saída Q̄ é realimentada na entrada D. Isso faz o flip-flop alternar entre 0 e 1 a cada pulso de clock.

```
Q̄ ──→ D
C  ← clock do sistema

Saídas:
  Q  → En_AC da ALU   (habilita gravação no AC no ciclo de execução)
  Q̄ → en do IR       (habilita gravação no IR no ciclo de busca)
```

---

## Como simular

### Requisitos

- [Digital](https://github.com/hneemann/Digital) — simulador de circuitos digitais (Java)

### Passos

1. Clone o repositório:
   ```bash
   git clone https://github.com/seu-usuario/cpu-8bits.git
   cd cpu-8bits
   ```

2. Abra o Digital e carregue o arquivo principal `cpu.dig`.

3. Programe a EEPROM com um dos exemplos abaixo (clique duas vezes na EEPROM para abrir o editor).

4. Clique em **Simulation → Play**.

5. Use o botão de clock manual para avançar passo a passo e observar o AC e MQ nos displays.

> **Dica:** a cada 2 pulsos de clock a CPU completa uma instrução. Use o clock manual para acompanhar o fetch e o execute separadamente.

---

## Exemplos de programas

### Exemplo 1 — Soma simples: `0 + 15 + 5 = 20`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0x0F` | N = 15 |
| `0x02` | `0x00` | ADD |
| `0x03` | `0x05` | N = 5 |

**Resultado:** AC = `0x14` (20)

---

### Exemplo 2 — Subtração: `30 - 8 = 22`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0x1E` | N = 30 |
| `0x02` | `0x20` | SUB |
| `0x03` | `0x08` | N = 8 |

**Resultado:** AC = `0x16` (22)

---

### Exemplo 3 — Multiplicação com overflow: `255 × 255 = 65025`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0xFF` | N = 255 |
| `0x02` | `0x40` | MUL |
| `0x03` | `0xFF` | N = 255 |

**Resultado:** AC = `0x01`, MQ = `0xFE` (65025 = 0xFE01)

---

### Exemplo 4 — Divisão: `17 ÷ 5`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0x11` | N = 17 |
| `0x02` | `0x60` | DIV |
| `0x03` | `0x05` | N = 5 |

**Resultado:** MQ = `3` (quociente), AC = `2` (resto)

---

### Exemplo 5 — Shift left: `1 << 1 = 2`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0x01` | N = 1 |
| `0x02` | `0x80` | SHL |
| `0x03` | `0x00` | N = ignorado |

**Resultado:** AC = `0x02` (2)

---

### Exemplo 6 — XOR: `0xAA XOR 0xFF = 0x55`

| Endereço | Valor | Instrução |
|---|---|---|
| `0x00` | `0x00` | ADD |
| `0x01` | `0xAA` | N = 170 |
| `0x02` | `0xE0` | XOR |
| `0x03` | `0xFF` | N = 255 |

**Resultado:** AC = `0x55` (85)

---

## Estrutura do repositório

```
cpu-8bits/
├── cpu.dig              # Circuito principal da CPU
├── ALU-completa.dig     # Subcircuito da ALU com AC e MQ
├── somador-8bits.dig    # Subcircuito do somador
├── README.md            # Este arquivo
└── exemplos/
    ├── soma.dig         # EEPROM programada com exemplo de soma
    ├── multiplicacao.dig
    └── divisao.dig
```

---

## Autor

Desenvolvido por **Maria Arielly** como atividade prática da disciplina de Arquitetura de Computadores.
