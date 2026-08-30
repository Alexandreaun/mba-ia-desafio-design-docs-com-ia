---
description: Criar um mapeamento modular da base de código para preparar a identificação de ADRs (Fase 1)
tags: [projeto, adr]
---

Execute o agente `adr-analyzer` para realizar a Fase 1: Mapeamento da Base de Código.

Isso irá:
1. Analisar toda a estrutura do projeto
2. Identificar tecnologias, frameworks e padrões arquiteturais
3. Dividir a base de código em módulos lógicos e independentes
4. Criar o arquivo `mapping.md` com a estrutura modular

O arquivo de mapeamento incluirá:
- Visão geral do projeto e stack tecnológica
- Módulos do sistema com IDs, estimativas de escopo e descrições
- Preocupações transversais (infraestrutura, autenticação, camada de dados, etc.)
- Diretrizes para a análise da Fase 2

**Uso**:
```
/adr-map [--project-dir=CAMINHO] [--context-dir=CAMINHO] [--output-dir=CAMINHO]
```

**Exemplos**:
```
/adr-map
# Mapeia o diretório atual e gera a saída em docs/adrs/mapping.md

/adr-map --project-dir=/caminho/para/projeto
# Mapeia um diretório de projeto específico

/adr-map --context-dir=docs/architecture
# Mapeia o diretório atual usando documentos/diagramas de arquitetura para embasar o mapeamento

/adr-map --project-dir=/legacy --context-dir=/legacy/docs --output-dir=analysis/adrs
# Controle total: diretórios personalizados de projeto, contexto e saída
```

**Arquivos de Contexto**: Suporte a todos os tipos de arquivo — documentos de arquitetura, documentos de design, arquivos README,
diagramas (PNG, SVG), imagens, PDFs e documentação da stack tecnológica. Eles ajudam o agente a compreender melhor
os limites dos módulos e os domínios de negócio.

**Saída**: `{OUTPUT_DIR}/mapping.md` (padrão: `docs/adrs/mapping.md`)

Após a conclusão da Fase 1, você pode usar o comando `/adr-identify` para identificar possíveis ADRs para módulos específicos.

Para bases de código grandes (mais de 10.000 arquivos), isso cria uma base para análise incremental sem sobrecarregar a janela de contexto. ---

## Instruções de Implementação

Quando o usuário invocar `/adr-map`:

Analise os argumentos para extrair:
- `--project-dir=<path>`: Diretório opcional a ser mapeado (padrão: `.` - diretório de trabalho atual)
- `--context-dir=<path>`: Diretório de contexto opcional contendo documentos/diagramas (padrão: nenhum)
- `--output-dir=<path>`: Diretório de saída opcional (padrão: `docs/adrs`)

Inicie o agente `adr-analyzer` usando a ferramenta Task:

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

**Com --context-dir:**
```
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código com --context-dir=docs/architecture"
```

**Com múltiplas opções:**
```
Task tool:
- subagent_type: adr-analyzer
- prompt: "Executar a Fase 1: Criar mapeamento da base de código com --project-dir=/legacy --context-dir=/legacy/docs --output-dir=analysis/adrs"
```

O agente irá:
1. Analisar o projeto em `--project-dir` (ou no diretório atual)
2. Carregar arquivos de contexto a partir de `--context-dir`, se fornecido (todos os tipos de arquivo)
3. Criar `{OUTPUT_DIR}/mapping.md` com a análise completa da base de código e notas de contexto opcionais