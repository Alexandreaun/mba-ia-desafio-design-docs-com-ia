---
description: Identificar possíveis ADRs para módulos específicos com base no mapeamento da base de código e transcrições de refinamento técnico (Fase 2)
tags: [project, adr]
---

Inicia o agente `adr-analyzer` para identificar possíveis ADRs nos módulos especificados, cruzando a análise técnica com as motivações de negócios contidas em arquivos de transcrição.

Quando múltiplos módulos são especificados, inicia agentes em paralelo para uma análise mais rápida.

**Pré-requisitos**: Execute `/adr-map` primeiro caso o arquivo `mapping.md` não exista no diretório de saída.

**O que faz**:
- Analisa o(s) módulo(s) especificado(s) com filtragem arquitetural rigorosa
- Lê arquivos de transcrição para correlacionar as decisões de código com as intenções e restrições discutidas nas reuniões de alinhamento técnico.
- Processa múltiplos módulos em paralelo quando 2 ou mais módulos são especificados
- Utiliza o histórico do Git internamente para enriquecer as possíveis ADRs com contexto temporal
- Cria arquivos ADR individuais em pastas organizadas por prioridade (sem números nos nomes dos arquivos)
- Integra naturalmente insights do Git e das transcrições ao conteúdo (datas, palavras-chave, evolução, motivações de negócio).
- Atualiza o índice com as descobertas

**Uso**:
```
/adr-identify [module-ids] [--output-dir=PATH] [--adrs-dir=PATH] [--transcript-file=PATH]
```

**Exemplos**:
```
/adr-identify
# Solicita quais módulos analisar (usa docs/adrs por padrão)

/adr-identify AUTH
# Analisa apenas o módulo AUTH

/adr-identify AUTH --transcript-file=docs/transcricao.md
# Analisa o módulo AUTH cruzando as evidências de código com as decisões extraídas da transcrição

/adr-identify AUTH ORDERS USERS --transcript-file=docs/transcricao.md
# Analisa múltiplos módulos em paralelo, garantindo que todos os agentes leiam a transcrição para embasar a pontuação e priorização das ADRs

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
- **Pontuação**: 3 dimensões (Escopo+Impacto, Custo de Mudança, Conhecimento da Equipe), total de 150 pontos (75 da pontuação base da Etapa 0 + 75 das dimensões). A presença de uma justificativa forte na transcrição aumenta automaticamente o peso do Escopo+Impacto.
- **Limiares**: ≥100 pts (67%, alta prioridade), 75-99 pts (50-66%, média prioridade), <75 pts (descartar).
- **Filtragem**: Etapa 0 (Identificação Positiva) + Sinais de Alerta (Red Flags) 1 a 5.
- **Integração Plural (Git + Transcrição)**: Enriquece o conteúdo combinando a evolução técnica (Git) com a intenção humana (Transcrição).

**Principais características**:
- **Etapa 0 (Identificação Positiva)**: Decisões tecnológicas fundamentais (serviços de infraestrutura, framework, ORM, API, autenticação, pagamento, IA/ML) acionam automaticamente a criação de uma ADR com pontuação base
- **Sinal de Alerta 5**: Consolida decisões excessivamente granulares em ADRs estratégicas mais abrangentes (evita a proliferação de ADRs)
- **Impacto Operacional**: Captura decisões relacionadas a observabilidade, resiliência e confiabilidade em produção

**Resultados esperados**:
- Apenas ~5% das descobertas se tornam ADRs (filtragem rigorosa)
- ADRs potenciais enriquecidas com o histórico do Git (datas, evolução, palavras-chave de intenção) e com as reais motivações de design da equipe.
- Sem seção separada de "Histórico do Git" ou "Resumo da Reunião" – insights integrados naturalmente ao conteúdo

**Nota**: Este processo identifica ADRs potenciais com base em evidências do código e de discussões registradas. Você decide quais delas documentar formalmente na Fase 3.

---

## Instruções de implementação

Quando o usuário invoca `/adr-identify` com IDs de módulo:

### Analisar Argumentos
Extrair do comando:
- IDs de módulo (ex.: AUTH, ORDERS, USERS)
- `--output-dir=<caminho>`: Diretório de saída opcional (padrão: `docs/adrs`)
- `--adrs-dir=<caminho>`: Diretório de ADRs existentes opcional (padrão: `{OUTPUT_DIR}/generated/`)
- `--transcript-file=<caminho>`: Arquivo de transcrição opcional contendo discussões de refinamento técnico

### Módulo Único (ex.: `/adr-identify AUTH`)
Iniciar um único agente `adr-analyzer` com a ferramenta Task. Se `--transcript-file` for fornecido, a instrução deve ser injetada no prompt:

**Sem --output-dir:**
```
Ferramenta de tarefa:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH"
```

**Com --transcript-file:**
Ferramenta de tarefa:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH. Leia o arquivo fornecido em --transcript-file para cruzar as restrições e intenções de design discutidas na reunião com a análise estrutural do código."


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

### Múltiplos Módulos (ex.: `/adr-identify AUTH DATA API --transcript-file=transcricao.md`)
Inicie múltiplos agentes `adr-analyzer` **em paralelo** usando uma única mensagem com múltiplas chamadas de ferramenta de tarefa. A instrução de leitura da transcrição DEVE ser passada individualmente para cada agente:

**Sem --output-dir:**
```
Mensagem única com múltiplas chamadas da ferramenta de tarefa:

Chamada da ferramenta de tarefa 1:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo AUTH. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido)."

Chamada da ferramenta de tarefa 2:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo ORDERS. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido)."

Chamada da ferramenta de tarefa 3:
- subagent_type: adr-analyzer
- prompt: "Identificar possíveis ADRs para o módulo PRODUCTS. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido)."
```

**With --output-dir:**
```
Mensagem única com múltiplas chamadas da ferramenta de tarefa:

Chamada da ferramenta de tarefa 1:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo AUTH com --output-dir=custom/path. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido) para cruzar as intenções da reunião com a análise do código."

Chamada da ferramenta de tarefa 2:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo ORDERS com --output-dir=custom/path. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido) para cruzar as intenções da reunião com a análise do código."

Chamada da ferramenta de tarefa 3:
- subagent_type: adr-analyzer
- prompt: "Identifique possíveis ADRs para o módulo PRODUCTS com --output-dir=custom/path. Considere as informações de negócios e de design contidas no arquivo definido em --transcript-file (se fornecido) para cruzar as intenções da reunião com a análise do código."
```

**IMPORTANTE**:
- Quando 2 ou mais módulos forem especificados, você DEVE enviar uma única mensagem com múltiplas chamadas da ferramenta Task
- Cada agente deve analisar apenas UM módulo específico, mas TODOS devem ler a mesma transcrição (se a flag existir) para unificar o contexto arquitetural.
- NÃO execute os agentes sequencialmente — execute-os em paralelo para obter melhor desempenho
- Cada agente criará seus próprios arquivos ADR potenciais nas pastas de prioridade apropriadas
- O arquivo de índice será atualizado por cada agente de forma independente
- Todos os agentes devem usar o mesmo `--output-dir` e `--transcript-file`, caso estes tenham sido especificados.

### Nenhum Módulo Especificado (ex.: `/adr-identify` ou `/adr-identify --output-dir=custom/path` ou `/adr-identify --transcript-file=docs/transcricao.md`)
1. Determine o diretório de saída (a partir de `--output-dir` ou do padrão `docs/adrs`)
2. Leia `{OUTPUT_DIR}/mapping.md` para obter os módulos disponíveis
3. Apresente a lista de módulos ao usuário
4. Pergunte qual(is) módulo(s) ele deseja analisar
5. Uma vez especificado(s), siga a lógica acima (único ou múltiplos), propagando as flags adicionais como `--transcript-file`.