---
description: Gerar ADRs formais a partir de ADRs potenciais
tags: [project, adr]
---

Inicia o agente `adr-generator` para gerar ADRs formais a partir de ADRs potenciais.

Quando vários módulos ou arquivos são especificados, inicia os agentes em paralelo para uma geração mais rápida.

**O que faz**:
- Gera ADRs formais utilizando o marcador XXX para a numeração
- Executa múltiplos alvos em paralelo quando dois ou mais alvos são especificados
- Detecta relações com ADRs existentes
- Organiza por módulo: `generated/{MODULE}/` ou `generated/{MODULE}/needs-input/`
- Requer renumeração manual após a geração

**Uso**:
```
/adr-generate [modules] [--context-dir=PATH] [--language=LOCALE] [--include-consider] [--output-dir=PATH]
```

**Exemplos**:
```
/adr-generate --all
# Gere TODOS os potenciais ADRs em docs/adrs/generated/{MODULE}/
# Excluindo a propriedade "consider" if --include-consider is not specified

/adr-generate CONFIG
# Gere em docs/adrs/generated/CONFIG/

/adr-generate CONFIG USERS ORDERS
# Gere múltiplos módulos em paralelo

/adr-generate --include-consider CONFIG
# Inclua ambos must-document AND consider priorities

/adr-generate --language=pt-BR --include-consider CONFIG USERS
# Gerar em português com todas as prioridades

/adr-generate --context-dir=docs/context/ CONFIG
# Gerar com contexto estratégico

/adr-generate --output-dir=output/adrs CONFIG
# Gerar para o diretório de saída personalizado: output/adrs/generated/CONFIG/
```

---

## Instruções de Implementação

Quando o usuário invoca `/adr-generate`:

### Passo 1: Identificar potenciais arquivos ADR

Analisar argumentos para obter:
- IDs de módulos (ex.: CONFIG, URSERS, ORDERS)
- Opções (--context-dir, --language, --include-consider, --output-dir)
- Definir diretório base de saída (padrão: `docs/adrs`, ou usar o valor de `--output-dir`)

**Comportamento padrão (sem --include-consider):**
Verificar apenas os itens de documentação obrigatória:
```
docs/adrs/potential-adrs/must-document/{MODULE}/*.md
```

**Com a flag --include-consider:**
Analise ambas as prioridades:
```
docs/adrs/potential-adrs/must-document/{MODULE}/*.md
docs/adrs/potential-adrs/consider/{MODULE}/*.md
```

Compile uma lista de TODOS os possíveis arquivos ADR em TODOS os módulos especificados.

### Passo 2: Iniciar agentes em paralelo

Inicie MÚLTIPLOS agentes `adr-generator` **em paralelo** usando uma ÚNICA mensagem com MÚLTIPLAS chamadas de ferramenta de tarefa — UM agente para cada possível arquivo ADR.

**Exemplo: `/adr-generate CONFIG ORDERS`**

Se CONFIG tiver 3 arquivos e ORDERS tiver 2 arquivos, iniciar 5 agentes:

```
Menssagem única com 5 chamadas de ferramenta de tarefa:

Task 1:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/must-document/CONFIG/payment-gateway.md"

Task 2:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/must-document/CONFIG/dual-paypal.md"

Task 3:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/consider/CONFIG/cache-strategy.md"

Task 4:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/must-document/ORDERS/rest-choice.md"

Task 5:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/must-document/ORDERS/grpc-internal.md"
```

### Com Opções

Inclua opções no prompt de cada agente:

```
Task 1:
- subagent_type: adr-generator
- prompt: "Gerar ADR formal a partir de docs/adrs/potential-adrs/must-document/BILLING/payment-gateway.md with --language=pt-BR and --context-dir=docs/context/"
```

**CRÍTICO**:
- É OBRIGATÓRIO enviar uma ÚNICA mensagem com MÚLTIPLAS chamadas da ferramenta Task
- UM agente por arquivo ADR potencial (NÃO um agente por módulo)
- TODOS os agentes são executados em paralelo
- Cada agente utiliza o marcador XXX para a numeração
- NÃO executar sequencialmente

### Nenhum módulo especificado (por exemplo, `/adr-generate`)
1. Analisar `docs/adrs/potential-adrs/` para identificar todos os módulos
2. Perguntar ao usuário quais módulos processar
3. Seguir os passos 1 e 2 acima