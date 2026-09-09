---
description: Gerar o RFC arquitetural formal do projeto a partir da transcrição, das ADRs já formalizadas e do código-fonte
tags: [project, rfc]
---

Inicia o agente `rfc-generator` para gerar o RFC arquitetural formal do projeto.

**O que faz**:
- Gera o RFC (padrão: `docs/RFC.md`) cruzando obrigatoriamente três fontes: `TRANSCRICAO.md`, as ADRs formais já geradas em `docs/adrs/` e o código-fonte real do projeto (`src/`, `prisma/schema.prisma`).
- Aplica a prioridade de fontes definida em `rfc-generator.md`: ADRs geradas → evidências no código → transcrição de reuniões.
- Distingue claramente propostas de decisões já confirmadas, citando o identificador da ADR (ex.: `ADR-004`) sempre que a proposta tocar em uma decisão já formalizada.
- Nunca inventa referências: toda citação de código aponta para um caminho real (`arquivo:linha`), toda citação de reunião aponta para um trecho real da transcrição (`[hh:mm] Nome`), toda decisão já confirmada é referenciada pelo identificador real da ADR.

**Uso**:
```
/rfc-generate [--transcript-file=PATH] [--adrs-dir=PATH] [--output=PATH] [--context-dir=PATH]
```

**Exemplos**:
```
/rfc-generate
# Gera docs/RFC.md usando TRANSCRICAO.md, docs/adrs/ e o código-fonte (src/, prisma/) como fontes padrão

/rfc-generate --transcript-file=docs/transcricao.md
# Usa um caminho customizado para a transcrição

/rfc-generate --context-dir=docs/context/
# Inclui documentos estratégicos complementares na análise
```

---

## Instruções de Implementação

Quando o usuário invocar `/rfc-generate`:

### Passo 1: Resolver as fontes obrigatórias

- `--transcript-file`: padrão `TRANSCRICAO.md` na raiz do projeto.
- `--adrs-dir`: padrão `docs/adrs/`. Listar todos os arquivos `ADR-*.md` presentes diretamente nesse diretório (ignorar `potential-adrs/` e `needs-input/`, que não são ADRs formais).
- Código-fonte: `src/` e `prisma/schema.prisma`. Sempre incluído por padrão, não requer flag.
- `--output`: padrão `docs/RFC.md`.
- `--context-dir` (opcional): documentos estratégicos complementares.

### Passo 2: Validar disponibilidade das fontes obrigatórias

Antes de iniciar o agente, verifique se `--transcript-file` existe e se `--adrs-dir` contém pelo menos uma ADR formal.

Se qualquer uma das duas estiver ausente, **não** inicie o agente: informe ao usuário quais fontes obrigatórias estão faltando e pergunte como prosseguir (ex.: apontar para outro caminho, ou confirmar que deseja gerar mesmo assim com rastreabilidade reduzida).

**CRÍTICO**:
- É proibido apresentar como "proposta em aberto" uma decisão que já está formalizada em uma ADR existente — deve ser citada como decisão confirmada, com o identificador.
- É proibido inventar caminho de arquivo, classe, endpoint ou trecho de código que não exista no código-fonte real.
- É proibido inventar ou aproximar um trecho da transcrição — toda citação de motivação de negócio ou debate técnico deve apontar para um timestamp real (`[hh:mm] Nome`) de `TRANSCRICAO.md`.
- Caso alguma dessas três fontes não sustente uma afirmação necessária, utilize `TBD` ou `QUESTÃO EM ABERTO`, conforme as regras do `rfc-generator.md` — nunca preencher a lacuna com suposição.

### Relatório

Ao final, confirme o caminho do arquivo gerado, quantas ADRs foram referenciadas, e quais seções ficaram marcadas com `QUESTÃO EM ABERTO` ou `TBD` por falta de evidência nas fontes.
