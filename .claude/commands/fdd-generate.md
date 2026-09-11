---
description: Gerar o FDD (Feature Design Doc) técnico do projeto a partir da RFC aprovada, das ADRs formais, da transcrição e do código-fonte
tags: [project, fdd]
---

Inicia o agente `fdd-generator` para gerar o FDD (Feature Design Doc) técnico do projeto, sem entrevista interativa — a geração é 100% orientada pelas fontes de dados já existentes no repositório.

**O que faz**:
- Gera o FDD (padrão: `docs/FDD.md`) cruzando obrigatoriamente quatro fontes, cada uma com um papel específico e não intercambiável: a **RFC aprovada** (`docs/RFC.md`, o coração do FDD — dita o "o quê" e o "como macro"), as **ADRs formais** já geradas em `docs/adrs/` (as regras do jogo, restrições inegociáveis), a **transcrição** em `TRANSCRICAO.md` (os casos de borda — exceções, receios da equipe, fallback) e o **código-fonte** real do projeto (a realidade tática — fronteiras de implementação e dependências reais).
- Aplica a prioridade de conflito definida em `fdd-generator.md`: ADRs (inegociáveis) → RFC (visão aprovada) → Código-fonte (realidade implementada) → Transcrição (contexto complementar).
- Nunca reabre uma alternativa que a RFC já descartou, nem questiona uma decisão já formalizada em ADR — apenas as detalha em nível de implementação.
- Gera as 12 seções obrigatórias do FDD, incluindo: Contratos Públicos com no mínimo 4 endpoints HTTP (cada um com payload de exemplo de requisição, payload de exemplo de resposta, headers e status codes); a Matriz de Erros no padrão `WEBHOOK_*` (reutilizando a hierarquia `AppError` existente); as Estratégias de Resiliência (citando a ADR de retry/backoff/DLQ quando existir); os 4 fluxos específicos do domínio de webhooks (criação na outbox, processamento pelo worker, retry, DLQ); e a seção "Integração com o Sistema Existente" com no mínimo 4 caminhos de arquivo reais.
- Nunca inventa referências: toda citação de código aponta para um caminho real (`arquivo:linha`), toda citação de reunião aponta para um trecho real da transcrição (`[hh:mm] Nome`), toda decisão já confirmada é referenciada pelo identificador real da ADR ou pela seção correspondente da RFC.

**Uso**:
```
/fdd-generate [--rfc-file=PATH] [--transcript-file=PATH] [--adrs-dir=PATH] [--output=PATH] [--context-dir=PATH]
```

**Exemplos**:
```
/fdd-generate
# Gera docs/FDD.md usando docs/RFC.md, docs/adrs/, TRANSCRICAO.md e o código-fonte (src/, prisma/) como fontes padrão

/fdd-generate --rfc-file=docs/RFC-webhooks.md
# Usa um caminho customizado para a RFC aprovada

/fdd-generate --output=docs/FDD-webhooks.md
# Gera em um caminho de saída customizado

/fdd-generate --context-dir=docs/context/
# Inclui documentos estratégicos complementares na análise
```

---

## Instruções de Implementação

Quando o usuário invocar `/fdd-generate`:

### Passo 1: Resolver as fontes obrigatórias

- `--rfc-file`: padrão `docs/RFC.md`.
- `--transcript-file`: padrão `TRANSCRICAO.md` na raiz do projeto.
- `--adrs-dir`: padrão `docs/adrs/`. Listar todos os arquivos `ADR-*.md` presentes diretamente nesse diretório (ignorar `potential-adrs/` e `needs-input/`, que não são ADRs formais).
- Código-fonte: `src/` e `prisma/schema.prisma`. Sempre incluído por padrão, não requer flag.
- `--output`: padrão `docs/FDD.md`.
- `--context-dir` (opcional): documentos estratégicos complementares.

### Passo 2: Validar disponibilidade das quatro fontes obrigatórias

Antes de iniciar o agente, verifique se `--rfc-file` existe, se `--transcript-file` existe, e se `--adrs-dir` contém pelo menos uma ADR formal.

Se qualquer uma das três estiver ausente, **não** inicie o agente: informe ao usuário quais fontes obrigatórias estão faltando e pergunte como prosseguir (ex.: apontar para outro caminho, gerar a RFC/ADRs primeiro com `/rfc-generate`/`/adr-generate`, ou confirmar que deseja gerar mesmo assim com rastreabilidade reduzida).

### Passo 3: Iniciar o agente `fdd-generator`

Inicie um único agente `fdd-generator` com a ferramenta Task, listando explicitamente as ADRs encontradas e informando as quatro fontes obrigatórias:

```
Ferramenta de tarefa:
- subagent_type: fdd-generator
- prompt: "Gere o FDD técnico em <OUTPUT>. É ESTRITAMENTE NECESSÁRIO basear o documento nas quatro fontes a seguir, respeitando o papel específico de cada uma, sem exceção:
  (1) a RFC aprovada em <RFC_FILE> — o coração do FDD. Utilize a Proposta Técnica, a Arquitetura, os Objetivos, o Contexto/Problema, o Fora do Escopo e a seção Impacto e Riscos como base macro para as seções correspondentes do FDD (Contexto, Objetivos, Escopo, Fluxos, Contratos, Riscos), detalhando essa visão em rotas, payloads exatos e contratos de integração técnica. Nunca reabra uma alternativa que a RFC já descartou;
  (2) TODAS as ADRs formais já existentes em <ADRS_DIR> (<lista de ADR-XXX encontradas>) — as regras do jogo, inegociáveis. Cite cada uma pelo identificador (ex.: ADR-004) sempre que um contrato, fluxo ou dependência implementar uma decisão já registrada; nunca sugira alternativa a uma decisão já formalizada. Em caso de conflito entre a RFC e uma ADR, a ADR prevalece;
  (3) o arquivo de transcrição em <TRANSCRIPT_FILE> — os casos de borda. Use-a como fonte primária para a Matriz de Erros e as Estratégias de Resiliência (exceções, receios da equipe, políticas de fallback, cenários como timeout de provedor externo), citando trechos no formato [hh:mm] Nome;
  (4) o código-fonte real do projeto (src/, prisma/schema.prisma) — a realidade tática, fonte primária e obrigatória da seção Dependências e Compatibilidade. Referencie caminhos de arquivo reais (arquivo:linha), nunca inventados.
  Gere as 12 seções obrigatórias do template, incluindo: em Contratos Públicos, no mínimo 4 endpoints HTTP distintos, cada um com payload de exemplo de requisição, payload de exemplo de resposta, headers e status codes documentados (se as fontes não sustentarem 4 endpoints reais ou hipotéticos plausíveis, declare a lacuna em vez de inventar); a Matriz de Erros com códigos no padrão WEBHOOK_* reutilizando a hierarquia AppError existente; os 4 fluxos específicos (criação do evento na outbox, processamento pelo worker, retry, DLQ); e a seção 'Integração com o Sistema Existente' com no mínimo 4 caminhos de arquivo reais e a descrição do ponto de integração de cada um.
  Se --context-dir=<PATH> foi fornecido, use também como fonte estratégica complementar."
```

**CRÍTICO**:
- É proibido apresentar como decisão em aberto algo que a RFC já propôs como solução ou que uma ADR já confirmou — a RFC e as ADRs são a base do "o quê"; o FDD só adiciona o "como" em detalhe de implementação.
- É proibido inventar caminho de arquivo, classe, endpoint, código de erro ou trecho de transcrição que não exista nas fontes reais.
- É proibido usar qualquer prefixo de código de erro que não seja `WEBHOOK_*` para erros específicos da feature.
- É proibido preencher "Contratos Públicos" com menos de 4 endpoints HTTP, ou com endpoints sem payload de exemplo de requisição, payload de exemplo de resposta e status codes.
- É proibido preencher "Integração com o Sistema Existente" com menos de 4 arquivos reais, ou com arquivos citados sem descrever o ponto de integração concreto.
- Caso alguma das quatro fontes não sustente uma afirmação necessária, utilize hipótese explícita (com 2-3 opções plausíveis) ou lacuna, conforme as regras do `fdd-generator.md` — nunca preencher com suposição silenciosa.

### Relatório

Ao final, confirme: o caminho do arquivo gerado; quantas ADRs foram referenciadas e quais; se a Proposta Técnica da RFC foi seguida integralmente ou exigiu ajuste; os 4+ endpoints HTTP documentados em "Contratos Públicos" (e, caso não tenha sido possível atingir o mínimo, por quê); os 4+ caminhos de arquivo reais citados em "Integração com o Sistema Existente"; e quais seções ficaram com hipóteses ou lacunas por falta de evidência nas fontes.
