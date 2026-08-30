---
description: Criar um mapeamento modular da base de código e de transcrição de refinamento para preparar a identificação de ADRs (Fase 1)
tags: [projeto, adr]
---

Execute o agente `adr-analyzer` para realizar a Fase 1: Mapeamento da Base de Código e Extração de Requisitos.

Isso irá:
1. Analisar toda a estrutura do projeto
2. Ler e interpretar arquivos de transcrição de reuniões (ex: transcricao.md) para capturar intenções arquiteturais, restrições e decisões de negócio.
3. Identificar tecnologias, frameworks e padrões arquiteturais presentes no código e discutidos na transcrição.
4. Dividir a base de código em módulos lógicos e independentes, cruzando a realidade do código existente com as novas funcionalidades refinadas.
5. Criar o arquivo `mapping.md` com a estrutura modular e as potenciais decisões mapeadas.

O arquivo de mapeamento incluirá:
- Visão geral do projeto, stack tecnológica e um resumo executivo das decisões extraídas da reunião de refinamento.
- Módulos do sistema com IDs, estimativas de escopo e descrições incluindo o impacto arquitetural da nova funcionalidade debatida.
- Preocupações transversais (infraestrutura, autenticação, camada de dados, etc.)
- Diretrizes e potenciais ADRs identificadas preliminarmente a partir da transcrição para embasar a análise da Fase 2

**Uso**:
```
/adr-map [--project-dir=CAMINHO] [--context-dir=CAMINHO] [--transcript-file=CAMINHO] [--output-dir=CAMINHO]
```

**Exemplos**:
```
/adr-map --transcript-file=transcricao.md
# Mapeia o diretório atual e cruza a análise de código com as decisões do arquivo transcricao.md e gera a saída em docs/adrs/mapping.md

/adr-map --project-dir=/caminho/para/projeto
# Mapeia um diretório de projeto específico

/adr-map --context-dir=docs/architecture
# Mapeia o diretório atual usando documentos/diagramas de arquitetura para embasar o mapeamento

/adr-map --project-dir=/legacy --context-dir=/legacy/docs --output-dir=analysis/adrs
# Controle total: diretórios personalizados de projeto, contexto e saída
```

**Arquivos de Contexto e Transcrição**: Suporte a todos os tipos de arquivo — documentos de arquitetura, documentos de design, arquivos README,
diagramas (PNG, SVG), imagens, PDFs e documentação da stack tecnológica. O uso da flag `--transcript-file` é crucial para que o agente compreenda o contexto humano, as motivações de negócios e as decisões arquiteturais tomadas antes mesmo de se tornarem código. Eles ajudam o agente a compreender melhor os limites dos módulos e os domínios de negócio.

**Saída**: `{OUTPUT_DIR}/mapping.md` (padrão: `docs/adrs/mapping.md`)

Após a conclusão da Fase 1, você pode usar o comando `/adr-identify` para identificar possíveis ADRs para módulos específicos. Para bases de código grandes (mais de 10.000 arquivos), isso cria uma base para análise incremental sem sobrecarregar a janela de contexto. 

---

## Instruções de Implementação

Quando o usuário invocar `/adr-map`:

Analise os argumentos para extrair:
- `--project-dir=<path>`: Diretório opcional a ser mapeado (padrão: `.` - diretório de trabalho atual)
- `--context-dir=<path>`: Diretório de contexto opcional contendo documentos/diagramas (padrão: nenhum)
- `--transcript-file=<path>`: Arquivo específico de transcrição de reunião (ex: transcricao.md) para extração de contexto e decisões técnicas (padrão: nenhum)
- `--output-dir=<path>`: Diretório de saída opcional (padrão: `docs/adrs`)

Inicie o agente `adr-analyzer` usando a ferramenta Task. Você deve incluir explicitamente a instrução de análise da transcrição no prompt da Task:

**Sem opções:**
```
Task tool:
- subagent_type: adr-analyzer
- prompt: "Execute a Fase 1: Crie o mapeamento da base de código."
```

**Com --project-dir:**
```
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código com --project-dir=/path/to/project"
```

**Com --transcript-file:**
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código. Leia atentamente o arquivo definido em --transcript-file, extraia as intenções de design, restrições e requisitos de negócio discutidos na reunião, e cruze essas informações com a análise estática do código para gerar o mapping.md."

**Com --context-dir:**
```
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código com --context-dir=docs/architecture"
```

**Com múltiplas opções:**
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código com --project-dir=/legacy --context-dir=/legacy/docs --transcript-file=transcricao.md --output-dir=analysis/adrs. Garanta que o mapeamento reflita as decisões técnicas e as motivações extraídas da transcrição, integrando-as à arquitetura identificada no código."

O agente irá:
1. Analisar o projeto em `--project-dir` (ou no diretório atual)
2. Ler, analisar e extrair as decisões arquiteturais preliminares do arquivo fornecido em `--transcript-file`.
3. Carregar arquivos de contexto a partir de `--context-dir`, se fornecido (todos os tipos de arquivo)
4. Criar `{OUTPUT_DIR}/mapping.md` com a análise completa da base de código, unificando a realidade estrutural com as diretrizes humanas da transcrição.