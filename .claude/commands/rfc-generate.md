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
- Preenche o campo **Revisores** com os participantes reais da reunião (extraídos de `TRANSCRICAO.md`) e o **Resumo Executivo (TL;DR)** de forma concisa.
- Documenta a **Proposta Técnica** em nível de visão geral, sem duplicar o detalhamento de implementação que pertence ao FDD.
- Garante no mínimo 2 **Alternativas Consideradas** reais (discutidas e descartadas na reunião, com o trade-off do descarte) e no mínimo 2 **Questões em Aberto** reais (levantadas na reunião e não decididas/adiadas).
- Documenta **Impacto e Riscos** (quem/o que é afetado, além dos riscos e mitigações) e as **Decisões Relacionadas** (ADRs formais aplicáveis).

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

### Passo 3: Iniciar o agente `rfc-generator`

Inicie um único agente `rfc-generator` com a ferramenta Task, listando explicitamente as ADRs encontradas e informando as três fontes obrigatórias:

```
Ferramenta de tarefa:
- subagent_type: rfc-generator
- prompt: "Gere o RFC arquitetural em <OUTPUT>. É ESTRITAMENTE NECESSÁRIO basear o documento nas três fontes a seguir, sem exceção:
  (1) o arquivo de transcrição em <TRANSCRIPT_FILE> — cite trechos relevantes no formato [hh:mm] Nome. Preencha o campo Revisores com os participantes reais da reunião extraídos dessa transcrição;
  (2) TODAS as ADRs formais já existentes em <ADRS_DIR> (<lista de ADR-XXX encontradas>) — referencie cada uma pelo identificador (ex.: ADR-004) sempre que a proposta tocar em uma decisão já confirmada, preenchendo o campo 'ADRs Relacionadas' do cabeçalho e a seção 8 'ADRs Relacionadas e Decisões Já Confirmadas'; nunca redescreva como proposta em aberto uma decisão que já tem ADR;
  (3) o código-fonte real do projeto (src/, prisma/schema.prisma) — referencie caminhos de arquivo reais (arquivo:linha), nunca inventados.
  Garanta no mínimo 2 Alternativas Consideradas reais (discutidas e descartadas na reunião, cada uma com o trade-off do descarte) e no mínimo 2 Questões em Aberto reais (levantadas na reunião e não decididas/adiadas). Mantenha a Proposta Técnica em nível de visão geral, sem duplicar detalhe de implementação do FDD. Documente Impacto e Riscos (quem/o que é afetado, além dos riscos e mitigações).
  Se --context-dir=<PATH> foi fornecido, use também como fonte estratégica complementar."
```

**CRÍTICO**:
- É proibido apresentar como "proposta em aberto" uma decisão que já está formalizada em uma ADR existente — deve ser citada como decisão confirmada, com o identificador.
- É proibido inventar caminho de arquivo, classe, endpoint ou trecho de código que não exista no código-fonte real.
- É proibido inventar ou aproximar um trecho da transcrição — toda citação de motivação de negócio ou debate técnico deve apontar para um timestamp real (`[hh:mm] Nome`) de `TRANSCRICAO.md`.
- É proibido inventar participantes para o campo Revisores — use somente nomes reais encontrados na transcrição; na ausência, `TBD`.
- É proibido preencher Alternativas Consideradas ou Questões em Aberto com itens inventados apenas para atingir o mínimo de 2 — se a transcrição não sustentar esse mínimo, declare a lacuna explicitamente em vez de inventar.
- Caso alguma dessas fontes não sustente uma afirmação necessária, utilize `TBD` ou `QUESTÃO EM ABERTO`, conforme as regras do `rfc-generator.md` — nunca preencher a lacuna com suposição.

### Relatório

Ao final, confirme: o caminho do arquivo gerado; quantas ADRs foram referenciadas; quem foi listado em Revisores; se os mínimos de 2 Alternativas Consideradas e 2 Questões em Aberto reais foram atingidos (e, caso não, por quê); e quais seções ficaram marcadas com `QUESTÃO EM ABERTO` ou `TBD` por falta de evidência nas fontes.
