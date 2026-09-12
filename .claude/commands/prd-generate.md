---
description: Gerar o PRD (Product Requirements Document) do projeto a partir da transcrição, das ADRs formais, da RFC aprovada, do FDD técnico e do código-fonte
tags: [project, prd]
---

Inicia o agente `prd-generator` para gerar o PRD (Product Requirements Document) de produto/negócio do projeto, sem entrevista interativa — a geração é 100% orientada pelas fontes de dados já existentes no repositório.

**O que faz**:
- Gera o PRD (padrão: `docs/PRD.md`) cruzando obrigatoriamente cinco fontes, cada uma com um papel específico e não intercambiável: a **transcrição** em `TRANSCRICAO.md` (fonte primária de contexto de negócio — problema original, motivação, público-alvo, cenários de uso e itens descartados/adiados na reunião), a **RFC aprovada** (`docs/RFC.md`, a visão macro já validada do problema, objetivos, escopo e alternativas descartadas), as **ADRs formais** já geradas em `docs/adrs/` (decisões técnicas confirmadas e inegociáveis), o **FDD** (`docs/FDD.md`, o detalhamento técnico do qual se traduzem critérios de aceitação e riscos de produto) e o **código-fonte** real do projeto (a realidade tática — o que já existe versus o que é novo).
- Aplica a prioridade de conflito definida em `prd-generator.md`: ADRs (inegociáveis) → RFC aprovada (visão validada) → FDD (detalhe técnico refinado) → Código-fonte (realidade implementada) → Transcrição (contexto de negócio bruto), com a transcrição tratada como fonte primária para conteúdo de negócio que os documentos técnicos normalmente não cobrem.
- Nunca reabre uma alternativa que a RFC já descartou, nem questiona uma decisão já formalizada em ADR — apenas as traduz para linguagem de produto.
- Gera as 12 seções obrigatórias do PRD (Resumo e contexto, Problema e motivação, Público-alvo e cenários de uso, Objetivos e métricas, Escopo, Requisitos funcionais, Requisitos não funcionais, Decisões e trade-offs principais, Dependências, Riscos e mitigação, Critérios de aceitação, Estratégia de testes e validação), garantindo no mínimo: 8 requisitos funcionais distintos discutidos na reunião e/ou formalizados na RFC/FDD; 1 objetivo com métrica e meta quantitativa; 2 itens em "Fora de escopo" explicitamente descartados ou adiados na reunião; e 2 riscos, cada um com probabilidade, impacto e mitigação.
- Nunca inventa referências: toda citação de código aponta para um caminho real (`arquivo:linha`), toda citação de reunião aponta para um trecho real da transcrição (`[hh:mm] Nome`), toda decisão já confirmada é referenciada pelo identificador real da ADR ou pela seção correspondente da RFC/FDD.

**Uso**:
```
/prd-generate [--transcript-file=PATH] [--adrs-dir=PATH] [--rfc-file=PATH] [--fdd-file=PATH] [--output=PATH] [--context-dir=PATH]
```

**Exemplos**:
```
/prd-generate
# Gera docs/PRD.md usando TRANSCRICAO.md, docs/adrs/, docs/RFC.md, docs/FDD.md e o código-fonte (src/, prisma/) como fontes padrão

/prd-generate --rfc-file=docs/RFC-webhooks.md --fdd-file=docs/FDD-webhooks.md
# Usa caminhos customizados para a RFC aprovada e o FDD

/prd-generate --output=docs/PRD-webhooks.md
# Gera em um caminho de saída customizado

/prd-generate --context-dir=docs/context/
# Inclui documentos estratégicos complementares na análise
```

---

## Instruções de Implementação

Quando o usuário invocar `/prd-generate`:

### Passo 1: Resolver as fontes obrigatórias

- `--transcript-file`: padrão `TRANSCRICAO.md` na raiz do projeto.
- `--adrs-dir`: padrão `docs/adrs/`. Listar todos os arquivos `ADR-*.md` presentes diretamente nesse diretório (ignorar `potential-adrs/` e `needs-input/`, que não são ADRs formais).
- `--rfc-file`: padrão `docs/RFC.md`.
- `--fdd-file`: padrão `docs/FDD.md`.
- Código-fonte: `src/` e `prisma/schema.prisma`. Sempre incluído por padrão, não requer flag.
- `--output`: padrão `docs/PRD.md`.
- `--context-dir` (opcional): documentos estratégicos complementares.

### Passo 2: Validar disponibilidade das cinco fontes obrigatórias

Antes de iniciar o agente, verifique se `--transcript-file` existe, se `--adrs-dir` contém pelo menos uma ADR formal, se `--rfc-file` existe e se `--fdd-file` existe.

Se qualquer uma dessas fontes estiver ausente, **não** inicie o agente: informe ao usuário quais fontes obrigatórias estão faltando e pergunte como prosseguir (ex.: apontar para outro caminho, gerar a RFC/ADRs/FDD primeiro com `/rfc-generate`, `/adr-generate` ou `/fdd-generate`, ou confirmar que deseja gerar mesmo assim com rastreabilidade reduzida).

### Passo 3: Iniciar o agente `prd-generator`

Inicie um único agente `prd-generator` com a ferramenta Task, listando explicitamente as ADRs encontradas e informando as cinco fontes obrigatórias:

```
Ferramenta de tarefa:
- subagent_type: prd-generator
- prompt: "Gere o PRD de produto em <OUTPUT>. É ESTRITAMENTE NECESSÁRIO basear o documento nas cinco fontes a seguir, respeitando o papel específico de cada uma, sem exceção:
  (1) o arquivo de transcrição em <TRANSCRIPT_FILE> — fonte primária de contexto de negócio: problema original, motivação, público-alvo, cenários de uso, objetivos/métricas discutidos e itens explicitamente descartados ou adiados durante a reunião (obrigatórios em 'Fora de escopo', com no mínimo 2 itens). Cite trechos no formato [hh:mm] Nome;
  (2) a RFC aprovada em <RFC_FILE> — a visão macro já validada. Utilize o Problema, os Objetivos, o Escopo/Fora do Escopo, os Requisitos (Funcionais e Não Funcionais) e as Alternativas descartadas como base para as seções correspondentes do PRD. Nunca reabra uma alternativa que a RFC já descartou;
  (3) TODAS as ADRs formais já existentes em <ADRS_DIR> (<lista de ADR-XXX encontradas>) — decisões técnicas confirmadas e inegociáveis. Cite cada uma pelo identificador (ex.: ADR-004) na seção 'Decisões e trade-offs principais' sempre que uma decisão já estiver formalizada; nunca a redescreva como proposta em aberto;
  (4) o FDD em <FDD_FILE> — o detalhamento técnico de implementação. Traduza os Critérios de Aceite Técnicos em critérios de aceitação de produto e os riscos técnicos específicos em riscos de produto, sem repetir o nível de detalhe do FDD (payloads, contratos, matriz de erros);
  (5) o código-fonte real do projeto (src/, prisma/schema.prisma) — a realidade tática: confirma requisitos já existentes versus novos e fundamenta dependências técnicas reais. Referencie caminhos de arquivo reais (arquivo:linha), nunca inventados.
  Gere as 12 seções obrigatórias do template (Resumo e contexto, Problema e motivação, Público-alvo e cenários de uso, Objetivos e métricas, Escopo, Requisitos funcionais, Requisitos não funcionais, Decisões e trade-offs principais, Dependências, Riscos e mitigação, Critérios de aceitação, Estratégia de testes e validação), garantindo os mínimos: no mínimo 8 requisitos funcionais distintos discutidos na reunião e/ou formalizados na RFC/FDD; no mínimo 1 objetivo com métrica e meta quantitativa; no mínimo 2 itens em 'Fora de escopo' rastreáveis à transcrição; e no mínimo 2 riscos, cada um com probabilidade, impacto e mitigação.
  Se --context-dir=<PATH> foi fornecido, use também como fonte estratégica complementar."
```

**CRÍTICO**:
- É proibido apresentar como decisão em aberto algo que a RFC já propôs como solução ou que uma ADR já confirmou.
- É proibido inventar caminho de arquivo, classe, endpoint ou trecho de transcrição que não exista nas fontes reais.
- É proibido duplicar o nível de detalhe de implementação da RFC/FDD (arquitetura, payloads, contratos) — o PRD fala em altitude de produto/negócio.
- É proibido preencher "Requisitos funcionais" com menos de 8 itens, "Fora de escopo" com menos de 2 itens, "Objetivos e métricas" sem nenhuma meta quantitativa, ou "Riscos e mitigação" com menos de 2 riscos, apenas para simular completude — se as fontes não sustentarem esses mínimos, declare a lacuna explicitamente.
- Caso alguma das cinco fontes não sustente uma afirmação necessária, utilize hipótese explícita (com 2-3 opções plausíveis) ou lacuna, conforme as regras do `prd-generator.md` — nunca preencher com suposição silenciosa.

### Relatório

Ao final, confirme: o caminho do arquivo gerado; quantas ADRs foram referenciadas e quais; se a RFC aprovada e o FDD foram utilizados integralmente ou exigiram hipóteses adicionais; a contagem de requisitos funcionais identificados (e se atingiu o mínimo de 8); os itens listados em "Fora de escopo" e o trecho de `TRANSCRICAO.md` que fundamenta cada um; e quais seções ficaram com hipóteses ou lacunas por falta de evidência nas fontes.
