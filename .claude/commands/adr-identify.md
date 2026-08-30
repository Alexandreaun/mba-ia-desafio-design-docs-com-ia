---
description: Identificar possíveis ADRs para módulos específicos com base no mapeamento da base de código (Fase 2)
tags: [project, adr]
---

Inicia o agente `adr-analyzer` para identificar possíveis ADRs nos módulos especificados.

Quando múltiplos módulos são especificados, inicia agentes em paralelo para uma análise mais rápida.

**Pré-requisitos**: Execute `/adr-map` primeiro caso o arquivo `mapping.md` não exista no diretório de saída.

**O que faz**:
- Analisa o(s) módulo(s) especificado(s) com filtragem arquitetural rigorosa
- Processa múltiplos módulos em paralelo quando 2 ou mais módulos são especificados
- Utiliza o histórico do Git internamente para enriquecer as possíveis ADRs com contexto temporal
- Cria arquivos ADR individuais em pastas organizadas por prioridade (sem números nos nomes dos arquivos)
- Integra naturalmente insights do Git ao conteúdo (datas, palavras-chave, evolução)
- Atualiza o índice com as descobertas

**Uso**:
```
/adr-identify [module-ids] [--output-dir=PATH] [--adrs-dir=PATH]
```

**Exemplos**:
```
/adr-identify
# Solicita quais módulos analisar (usa docs/adrs por padrão)

/adr-identify AUTH
# Analisa apenas o módulo AUTH

/adr-identify AUTH ORDERS USERS
# Analisa múltiplos módulos

/adr-identify --output-dir=output/adrs AUTH
# Analisa o módulo AUTH com um diretório de saída personalizado

/adr-identify --adrs-dir=docs/adrs/generated AUTH
# Usa ADRs existentes de um diretório personalizado para contexto e detecção de duplicatas
```

**Estrutura de saída**:
```
{OUTPUT_DIR}/potential-adrs/          # Default: docs/adrs/potential-adrs
├── must-document/    # High priority (score ≥100 out of 150)
│   └── MODULE-ID/
│       ├── decision-title-kebab-case.md
│       └── another-architectural-decision.md
└── consider/         # Medium priority (score 75-99 out of 150)
    └── MODULE-ID/
        └── medium-priority-decision.md
```

**Importante**: Os nomes de arquivo utilizam o formato *kebab-case* descritivo, SEM números. A numeração ocorre na Fase 3, quando os ADRs formais são gerados.

**Integração com ADRs existentes**: Varre automaticamente o diretório `{OUTPUT_DIR}/generated/` (ou o caminho especificado em `--adrs-dir`) em busca de ADRs existentes para:
- Evitar identificação de duplicatas (aviso de similaridade >70%)
- Detectar relações com decisões existentes (similaridade entre 40% e 70%)
- Fornecer contexto de linha do tempo (evolução da decisão, padrões de substituição)
- Sugerir oportunidades de consolidação (detalhes de implementação de ADRs existentes)

**Como funciona**:
- **Pontuação**: 3 dimensões (Escopo+Impacto, Custo de Mudança, Conhecimento da Equipe), total de 150 pontos (75 da pontuação base da Etapa 0 + 75 das dimensões)
- **Limiares**: ≥100 pts (67%, alta prioridade), 75-99 pts (50-66%, média prioridade), <75 pts (descartar)
- **Filtragem**: Etapa 0 (Identificação Positiva) + Sinais de Alerta (Red Flags) 1 a 5
- **Integração com Git**: Enriquece o conteúdo com contexto temporal (datas, evolução, palavras-chave de intenção)

**Principais características**:
- **Etapa 0 (Identificação Positiva)**: Decisões tecnológicas fundamentais (serviços de infraestrutura, framework, ORM, API, autenticação, pagamento, IA/ML) acionam automaticamente a criação de uma ADR com pontuação base
- **Sinal de Alerta 5**: Consolida decisões excessivamente granulares em ADRs estratégicas mais abrangentes (evita a proliferação de ADRs)
- **Impacto Operacional**: Captura decisões relacionadas a observabilidade, resiliência e confiabilidade em produção

**Resultados esperados**:
- Apenas ~5% das descobertas se tornam ADRs (filtragem rigorosa)
- ADRs potenciais enriquecidas com o histórico do Git (datas, evolução, palavras-chave de intenção)
- Sem seção separada de "Histórico do Git" – insights integrados naturalmente ao conteúdo

**Nota**: Este processo identifica ADRs potenciais com base em evidências. Você decide quais delas documentar formalmente na Fase 3.

---

## Instruções de implementação

Quando o usuário invoca `/adr-identify` com IDs de módulo:

### Analisar Argumentos
Extrair do comando:
- IDs de módulo (ex.: AUTH, DATA, API)
- `--output-dir=<caminho>`: Diretório de saída opcional (padrão: `docs/adrs`)
- `--adrs-dir=<caminho>`: Diretório de ADRs existentes opcional (padrão: `{OUTPUT_DIR}/generated/`)

### Módulo Único (ex.: `/adr-identify AUTH`)
Iniciar um único agente `adr-analyzer` com a ferramenta Task:

**Sem --output-dir:**
```
Ferramenta de tarefa:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH"
```

**Com --output-dir:**
```
Ferramenta de tarefa:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH com --output-dir=custom/path"
```

**Com --adrs-dir:**
```
Ferramenta de tarefa:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH com --adrs-dir=docs/adrs/generated"
```

### Múltiplos Módulos (por exemplo, `/adr-identify AUTH DATA API` ou `/adr-identify --output-dir=output/adrs AUTH DATA`)
Inicie múltiplos agentes `adr-analyzer` **em paralelo** usando uma única mensagem com múltiplas chamadas de ferramenta de tarefa:

**Sem --output-dir:**
```
Mensagem única com múltiplas chamadas da ferramenta de tarefa:

Chamada da ferramenta de tarefa 1:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo AUTH"

Chamada da ferramenta de tarefa 2:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo DATA"

Chamada da ferramenta de tarefa 3:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo API"
```

**With --output-dir:**
```
Mensagem única com múltiplas chamadas da ferramenta de tarefa:

Chamada da ferramenta de tarefa 1:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH com --output-dir=custom/path"

Chamada da ferramenta de tarefa 2:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo DATA com --output-dir=custom/path"

Chamada da ferramenta de tarefa 3:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo API com --output-dir=custom/path"
```

**IMPORTANTE**:
- Quando 2 ou mais módulos forem especificados, você DEVE enviar uma única mensagem com múltiplas chamadas da ferramenta Task
- Cada agente deve analisar apenas UM módulo específico
- NÃO execute os agentes sequencialmente — execute-os em paralelo para obter melhor desempenho
- Cada agente criará seus próprios arquivos ADR potenciais nas pastas de prioridade apropriadas
- O arquivo de índice será atualizado por cada agente de forma independente
- Todos os agentes devem usar o mesmo `--output-dir`, caso este tenha sido especificado

### Nenhum Módulo Especificado (ex.: `/adr-identify` ou `/adr-identify --output-dir=custom/path`)
1. Determine o diretório de saída (a partir de `--output-dir` ou do padrão `docs/adrs`)
2. Leia `{OUTPUT_DIR}/mapping.md` para obter os módulos disponíveis
3. Apresente a lista de módulos ao usuário
4. Pergunte qual(is) módulo(s) ele deseja analisar
5. Uma vez especificado(s), siga a lógica acima (único ou múltiplos)