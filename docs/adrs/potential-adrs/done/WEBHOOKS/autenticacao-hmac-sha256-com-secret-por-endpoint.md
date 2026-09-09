# ADR em Potencial: Autenticação de Webhooks via HMAC-SHA256 com Secret por Endpoint e Rotação

**Módulo**: WEBHOOKS
**Categoria**: Segurança (Autenticação de Integração Externa)
**Prioridade**: Obrigatório Documentar (Pontuação: 130/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

A equipe (liderada pela engenheira de segurança Sofia) decidiu que cada evento de webhook enviado a um cliente será assinado com **HMAC-SHA256**, permitindo ao cliente verificar autenticidade e integridade do payload. A assinatura vai no header `X-Signature`. Diferente de uma secret global da plataforma, **cada endpoint de webhook cadastrado por cliente possui sua própria secret única** — decisão explicitamente justificada por um incidente real já vivido pela equipe ("a gente já teve cliente que vazou secret em log de aplicação dele uma vez", `[09:22]` Diego). A secret é **rotacionável via API**, com a secret antiga permanecendo válida por um **grace period de 24 horas** em paralelo à nova, para permitir migração sem downtime do lado do cliente.

Esta é a primeira decisão de segurança de integração externa (outbound) do projeto — hoje o sistema só possui autenticação interna via JWT stateless para usuários operadores/admins. Como o módulo webhooks ainda não existe no código, toda a evidência vem de `TRANSCRICAO.md`; não há geração/armazenamento de secret nem lógica de assinatura HMAC implementadas hoje.

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: Define o modelo de confiança de toda a superfície de integração externa da plataforma — os três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) e quaisquer futuros consumidores de webhook dependerão deste mecanismo para confiar nos eventos recebidos.
- **Trade-offs**: Secret por endpoint (não global) foi escolhida deliberadamente para limitar o "blast radius" de um vazamento — trade-off de maior complexidade de armazenamento/rotação em troca de isolamento de incidente.
- **Complexidade**: Introduz gestão de ciclo de vida de secret (geração, rotação, grace period de validade dupla) — um problema que o projeto nunca precisou resolver antes (JWT hoje é stateless, sem rotação de segredo por entidade).
- **Conhecimento da Equipe**: Qualquer engenheiro implementando o CRUD de configuração de webhook precisa entender o fluxo de rotação com grace period; a própria Sofia reservou dias específicos de revisão de segurança dedicados a "HMAC e geração de secret" antes do deploy (`[09:46]`), sinalizando a criticidade do conhecimento.
- **Implicações Futuras**: Uma vez que clientes externos implementem verificação HMAC do lado deles, mudar o esquema de assinatura no futuro se torna uma mudança coordenada e potencialmente disruptiva para integrações de produção já em uso.
- **Contexto Temporal**: Decisão fechada integralmente nesta reunião, conduzida pela especialista de segurança da equipe; sem histórico de código anterior.

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/middlewares/auth.middleware.ts`](../../../../../src/middlewares/auth.middleware.ts) — único mecanismo de autenticação existente hoje no projeto (JWT stateless via `jsonwebtoken`), contraste relevante: é autenticação de usuário interno, não de assinatura de payload outbound; confirma que HMAC será um mecanismo novo, não uma extensão do JWT existente.
- `prisma/schema.prisma` — não contém (ainda) nenhum campo de `secret`/`webhookSecret` em nenhum modelo; confirma que armazenamento de secret por endpoint é inteiramente novo.
- `package.json` — não lista nenhuma dependência de criptografia além das nativas do Node (`jsonwebtoken`, `bcrypt`); o módulo nativo `crypto` do Node (não uma biblioteca externa) é a via natural para HMAC-SHA256, embora isso não tenha sido especificado explicitamente na reunião.

### Evidência no Código
```typescript
// src/middlewares/auth.middleware.ts — modelo de autenticação existente (contraste)
// Autenticação hoje é JWT stateless de usuário interno (ADMIN/OPERATOR),
// sem qualquer mecanismo de assinatura de payload para consumidores externos.
export const authenticate: RequestHandler = (req, _res, next) => {
  const token = /* ... extrai Bearer token ... */;
  const payload = jwt.verify(token, env.JWT_SECRET) as JwtPayload;
  req.user = { id: payload.sub, email: payload.email, role: payload.role };
  next();
};
```

### Evidência na Transcrição
- `[09:19]` Sofia: motivação do requisito — "O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio."
- `[09:20]` Sofia: escolha do padrão — "Padrão é HMAC. A gente assina o payload com uma secret compartilhada... manda a assinatura num header tipo X-Signature."
- `[09:20]` Sofia: escolha do algoritmo — "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."
- `[09:21]` Sofia: decisão de secret por endpoint (não global) — "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."
- `[09:21]` Sofia: decisão de rotação com grace period — "a secret tem que ser rotacionável... Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo... Depois disso, a antiga morre."
- `[09:22]` Diego: justificativa baseada em incidente real vivido pela equipe (vazamento de secret em log de cliente).
- `[09:22]` Sofia: fechamento formal — "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."
- `[09:46]` Sofia: reserva de dias de revisão de segurança dedicada especificamente a "HMAC e geração de secret" antes do deploy, reforçando a criticidade desta decisão.

### Análise de Impacto (Git)
- Histórico do Git não aplicável — nenhuma lógica de assinatura HMAC ou armazenamento de secret existe no repositório hoje. Único commit do projeto é `init repository`, sem evolução incremental a analisar.

### Alternativas (observáveis na transcrição)
- **Secret global da plataforma (compartilhada entre todos os endpoints/clientes)** — rejeitada implicitamente por Sofia em favor de secret por endpoint, justificada por limitar o impacto de um vazamento isolado.
- Nenhum algoritmo alternativo a HMAC-SHA256 foi discutido — a escolha foi apresentada e aceita diretamente como "padrão de mercado" (`[09:20]`), sem debate de alternativas (ex.: assinatura assimétrica).

## Questões a abordar na ADR (se criada)

- Qual problema estava sendo resolvido (autenticidade e integridade de payloads enviados a terceiros)?
- Por que HMAC-SHA256 com secret por endpoint em vez de secret global ou de um esquema assimétrico?
- Como funciona o fluxo de rotação (geração de nova secret, período de validade dupla de 24h, invalidação da antiga)?
- Quais são as consequências de longo prazo (necessidade de armazenamento seguro da secret no banco, requisito de revisão de segurança dedicada antes de cada deploy relacionado)?

## ADRs Potenciais Relacionadas
- Garantia de Entrega At-Least-Once com Idempotência via X-Event-Id (ambos compõem o conjunto de headers de segurança/idempotência enviados em cada requisição de webhook)

## Observações Adicionais

- TLS obrigatório (URL do webhook deve ser `https`) foi explicitamente classificado pela própria Sofia como *não sendo uma decisão arquitetural*, e sim uma validação de schema Zod (`[09:23]`: "Isso na verdade nem é decisão arquitetural, é só uma validação no schema Zod") — consolidado como detalhe de suporte dentro desta ADR, não como ADR separada (Sinal de Alerta 3).
- Limite de payload de 64KB com erro (sem truncar) em caso de excesso (`[09:23]-[09:24]`) foi explicitamente classificado por Larissa como requisito não funcional, não decisão arquitetural separada (`[09:24]`: "não vejo como decisão arquitetural separada, é só requisito não funcional") — mencionar apenas como nota de suporte, sem ADR própria.
- O header `X-Webhook-Id` (identificador do endpoint cadastrado, sugerido por Sofia em `[09:44]` para desambiguar múltiplos cadastros do mesmo cliente) é um detalhe de formato de headers, a consolidar na ADR de entrega/idempotência ou nesta, conforme a formalização escolhida — não como ADR separada.
