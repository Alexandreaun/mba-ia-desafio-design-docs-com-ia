---
description: Gerar o TRACKER de rastreabilidade cruzada do projeto a partir do PRD, da RFC, do FDD, das ADRs formais, da transcrição e do código-fonte
tags: [project, tracker]
---

Inicia o agente `tracker-generator` para gerar o TRACKER de rastreabilidade cruzada do projeto, sem entrevista interativa — a geração é 100% orientada pelos documentos e fontes já existentes no repositório.

**O que faz**:
- Gera o TRACKER (padrão: `docs/TRACKER.md`) inventariando todo item identificável em quatro documentos: **PRD** (`docs/PRD.md`), **RFC aprovada** (`docs/RFC.md`), **FDD** (`docs/FDD.md`) e **todas as ADRs formais** já geradas em `docs/adrs/`.
- Para cada item inventariado, busca a evidência real que o sustenta em exatamente uma de duas fontes: `TRANSCRICAO.md` (trecho no formato `[hh:mm] Nome`) ou o código-fonte real do projeto (`src/`, `prisma/schema.prisma`, caminho de arquivo real).
- Produz uma única tabela markdown com as colunas obrigatórias `ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização`, usando o esquema de identificadores definido em `tracker-generator.md` (ex.: `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-004`).
- Classifica cada item em rastreável (entra na tabela principal), sem rastreabilidade direta (hipótese/lacuna já assumida no próprio documento-fonte, listada à parte) ou divergente (o documento afirma algo que a transcrição/código não sustentam, listado em "Divergências Identificadas").
- Garante cobertura mínima de 80% dos itens identificáveis com linha correspondente na tabela principal; nunca força uma correspondência apenas para atingir esse número — se a cobertura ficar abaixo de 80%, declara explicitamente por que a evidência não foi encontrada.
- Nunca inventa referências: toda "Localização" aponta para um trecho real da transcrição ou um caminho real de arquivo, nunca aproximado ou suposto.

**Uso**:
```
/tracker-generate [--transcript-file=PATH] [--adrs-dir=PATH] [--rfc-file=PATH] [--fdd-file=PATH] [--prd-file=PATH] [--output=PATH]
```

**Exemplos**:
```
/tracker-generate
# Gera docs/TRACKER.md usando TRANSCRICAO.md, docs/adrs/, docs/RFC.md, docs/FDD.md, docs/PRD.md e o código-fonte (src/, prisma/) como fontes padrão

/tracker-generate --prd-file=docs/PRD-webhooks.md --rfc-file=docs/RFC-webhooks.md
# Usa caminhos customizados para o PRD e a RFC

/tracker-generate --output=docs/TRACKER-webhooks.md
# Gera em um caminho de saída customizado
```

---

## Instruções de Implementação

Quando o usuário invocar `/tracker-generate`:

### Passo 1: Resolver as fontes obrigatórias

- `--transcript-file`: padrão `TRANSCRICAO.md` na raiz do projeto.
- `--adrs-dir`: padrão `docs/adrs/`. Listar todos os arquivos `ADR-*.md` presentes diretamente nesse diretório (ignorar `mapping.md`, `potential-adrs/` e `potential-adrs-index.md`, que não são ADRs formais).
- `--rfc-file`: padrão `docs/RFC.md`.
- `--fdd-file`: padrão `docs/FDD.md`.
- `--prd-file`: padrão `docs/PRD.md`.
- Código-fonte: `src/` e `prisma/schema.prisma`. Sempre incluído por padrão, não requer flag.
- `--output`: padrão `docs/TRACKER.md`.

### Passo 2: Validar disponibilidade das cinco fontes obrigatórias

Antes de iniciar o agente, verifique se `--transcript-file` existe, se `--adrs-dir` contém pelo menos uma ADR formal, se `--rfc-file` existe, se `--fdd-file` existe e se `--prd-file` existe.

Se qualquer uma dessas fontes estiver ausente, **não** inicie o agente: informe ao usuário quais documentos obrigatórios estão faltando e pergunte como prosseguir (ex.: apontar para outro caminho, gerar o documento faltante primeiro com `/rfc-generate`, `/adr-generate`, `/fdd-generate` ou `/prd-generate`, ou confirmar que deseja gerar mesmo assim com cobertura reduzida).

### Passo 3: Iniciar o agente `tracker-generator`

Inicie um único agente `tracker-generator` com a ferramenta Task, listando explicitamente as ADRs encontradas e informando as cinco fontes obrigatórias:

```
Ferramenta de tarefa:
- subagent_type: tracker-generator
- prompt: "Gere o tracker de rastreabilidade em <OUTPUT>. É ESTRITAMENTE NECESSÁRIO inventariar itens dos quatro documentos a seguir e rastrear cada um até sua evidência real:
  DOCUMENTOS A RASTREAR:
  (1) o PRD em <PRD_FILE> — requisitos funcionais/não funcionais, objetivos/métricas, escopo/fora de escopo, decisões e trade-offs, dependências, riscos, critérios de aceitação;
  (2) a RFC aprovada em <RFC_FILE> — problema, objetivos, requisitos, restrições, alternativas consideradas, trade-offs, riscos, questões em aberto;
  (3) o FDD em <FDD_FILE> — objetivos técnicos, contratos públicos, matriz de erros WEBHOOK_*, estratégias de resiliência, dependências e compatibilidade, critérios de aceite técnicos, riscos, integração com o sistema existente;
  (4) TODAS as ADRs formais já existentes em <ADRS_DIR> (<lista de ADR-XXX encontradas>) — a decisão registrada em cada uma, e as alternativas/consequências quando agregarem rastreabilidade distinta.
  FONTES DE EVIDÊNCIA (para preencher Fonte e Localização, nunca outra coisa):
  (a) o arquivo de transcrição em <TRANSCRIPT_FILE> — Localização no formato [hh:mm] Nome;
  (b) o código-fonte real do projeto (src/, prisma/schema.prisma) — Localização como caminho real de arquivo, nunca inventado.
  Gere uma única tabela markdown com exatamente as colunas ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização, usando o esquema de identificadores definido em tracker-generator.md (ex.: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-004), ordenada por Documento (PRD → RFC → ADRs em ordem numérica → FDD) e pela ordem de aparição de cada item no documento original.
  Classifique cada item inventariado em: rastreável (linha na tabela principal), sem rastreabilidade direta (hipótese/lacuna já assumida no próprio documento-fonte, listada na seção correspondente, fora do denominador de cobertura), ou divergente (o documento afirma algo que a transcrição/código não sustentam, listado em 'Divergências Identificadas').
  Garanta cobertura mínima de 80% dos itens identificáveis (excluindo os já marcados como hipótese/lacuna nos documentos-fonte) com linha na tabela principal. Se a cobertura inicial ficar abaixo de 80%, aprofunde a busca de evidências antes de aceitar um item como não rastreável; só finalize abaixo de 80% se a evidência genuinamente não existir, declarando isso no relatório final."
```

**CRÍTICO**:
- É proibido inventar um timestamp de `TRANSCRICAO.md` ou um caminho de arquivo que não exista — declarar o item como não rastreável é sempre preferível a uma citação forçada.
- É proibido usar qualquer valor de "Fonte" diferente de `TRANSCRICAO` ou `CODIGO`.
- É proibido forçar linhas na tabela principal apenas para simular os 80% de cobertura.
- É proibido resumir múltiplos itens distintos em uma única linha para reduzir o volume da tabela — cada requisito, decisão, risco ou item individual recebe sua própria linha.
- É proibido omitir a seção "Divergências Identificadas" — se nenhuma divergência for encontrada, o documento deve declarar isso explicitamente em vez de omitir a seção.

### Relatório

Ao final, confirme: o caminho do arquivo gerado; o total de itens inventariados, quantos foram rastreados com sucesso e o percentual de cobertura atingido; quantos itens caíram em "Itens Sem Rastreabilidade Direta" e por quê; quantas divergências foram identificadas entre os documentos e as fontes reais; e, caso a cobertura mínima de 80% não tenha sido atingida, quais itens ficaram de fora e por quê.
