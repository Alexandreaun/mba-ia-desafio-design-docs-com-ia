# ADR em Potencial: Estratégia de Autenticação JWT Stateless com RBAC Embutido e Hash de Senha via bcrypt

**Módulo**: AUTH
**Categoria**: Segurança (Autenticação e Autorização)
**Prioridade**: Obrigatório Documentar (Pontuação: 140/150)
**Data de Identificação**: 2026-08-31

---

## O Que Foi Identificado

O módulo AUTH implementa o mecanismo de autenticação de todo o OMS: emissão de um JSON Web Token stateless no login (`AuthService.signToken`, `src/modules/auth/auth.service.ts:47-50`), assinado com um segredo compartilhado (`JWT_SECRET`, validado no boot via Zod com mínimo de 16 caracteres — `src/config/env.ts:8`) e expiração configurável (`JWT_EXPIRES_IN`, default `8h` — `src/config/env.ts:9`). O payload carrega `{ sub, email, role }`, ou seja, o RBAC de dois papéis (`ADMIN`/`OPERATOR`) fica embutido diretamente no claim do token, não em uma tabela de sessão ou permissões consultada em runtime. A verificação de senha usa `bcrypt.compare` (`auth.service.ts:36`) contra o hash armazenado por `UserService`. Não há refresh token, não há blacklist/revogação de token, não há sessão do lado do servidor — é um design deliberadamente stateless.

Esse mecanismo não é local ao módulo AUTH: o middleware `authenticate` (`src/middlewares/auth.middleware.ts:27-47`) verifica o JWT e popula `req.user`, e `requireRole(...roles)` (linhas 49-61) aplica o RBAC — ambos consumidos pelas rotas de USERS, CUSTOMERS, PRODUCTS e ORDERS (todo endpoint protegido do sistema hoje depende dessas duas funções). A análise já registrada do módulo MIDDLEWARES nesta mesma Fase 2 confirma essa separação de responsabilidade: "o mecanismo de autenticação em si (JWT, emissão/verificação de token) é decisão do módulo AUTH... o escopo de MIDDLEWARES é estritamente a camada de enforcement" (ver `potential-adrs-index.md`, Notas do Processo — MIDDLEWARES).

O histórico de Git não oferece granularidade temporal para esta decisão: o repositório foi criado em um único commit de inicialização (`7ef4317 init repository`, 2026-06-24 15:41:54 -0300), sem commits incrementais subsequentes tocando `auth.service.ts`, `auth.middleware.ts` ou `env.ts`. A implementação já nasce madura (schema de env validado, RBAC de dois papéis, hashing de senha) — não há evolução a analisar, apenas o estado final estável desde a inicialização do projeto, o que é consistente com o padrão já observado nos demais módulos analisados nesta Fase 2 (WEBHOOKS, ORDERS, SHARED, MIDDLEWARES, PRODUCTS).

## Por Que Isso Pode Merecer uma ADR

- **Impacto**: é a base de autenticação de todo o sistema — afeta diretamente 5 módulos (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS), a camada de enforcement em MIDDLEWARES, e será estendido ao futuro módulo WEBHOOKS (o endpoint de replay de DLQ exige role `ADMIN` via este mesmo JWT, confirmado em `[09:35]-[09:36]` da reunião).
- **Compromissos (Trade-offs)**: design stateless significa que não há como revogar um token antes da expiração — se uma credencial é comprometida ou o papel de um usuário muda, o token antigo continua válido até expirar naturalmente (até 8h por padrão). Não há refresh token: o usuário precisa reautenticar quando o token expira. Essas são consequências de segurança que uma ADR formal precisaria justificar e registrar como risco aceito.
- **Complexidade**: o RBAC de apenas 2 papéis é embutido diretamente no claim do JWT (não é consultado dinamicamente); trocar de mecanismo (sessão com store, IdP externo, refresh tokens) exigiria tocar em todas as rotas protegidas do sistema.
- **Conhecimento da Equipe**: qualquer engenheiro que crie um novo endpoint protegido precisa entender como `authenticate`/`requireRole` funcionam, o formato do payload do token e as limitações de revogação — conhecimento transversal a praticamente todo trabalho no backend.
- **Implicações Futuras**: o modelo binário ADMIN/OPERATOR pode não ser suficiente se features futuras exigirem permissões mais granulares (ex.: escopo por cliente B2B nos endpoints de configuração de webhook, mencionados na reunião em `[09:31]-[09:32]` como "endpoint autenticado normal" sem regra de acesso por `customer_id` ainda definida).
- **Contexto Temporal**: a ausência de qualquer evolução incremental no Git (implementação já completa desde a inicialização do repositório) e a reutilização acrítica do mecanismo na reunião de webhooks (sem nenhum questionamento sobre trocar ou estender o JWT) indicam uma decisão estável e consolidada — exatamente o tipo de decisão fundacional que corre risco de nunca ser documentada justamente por parecer "óbvia" ou "já dada".

## Evidências Encontradas na Base de Código

### Arquivos Principais
- [`src/modules/auth/auth.service.ts`](../../../../../src/modules/auth/auth.service.ts) - Linhas 1-51 - `login()`, `signToken()`, verificação de senha via `bcrypt.compare`, assinatura JWT com payload `{ sub, email, role }`
- [`src/middlewares/auth.middleware.ts`](../../../../../src/middlewares/auth.middleware.ts) - Linhas 1-61 - `authenticate` (verificação de JWT via `jwt.verify`) e `requireRole(...roles)` (RBAC), tipo `AuthUser`
- [`src/config/env.ts`](../../../../../src/config/env.ts) - Linhas 3-10 - validação de `JWT_SECRET` (mínimo 16 caracteres) e `JWT_EXPIRES_IN` (default `8h`) via Zod, fail-fast no boot
- [`src/modules/auth/auth.schemas.ts`](../../../../../src/modules/auth/auth.schemas.ts) - Linhas 1-16 - `registerSchema`/`loginSchema`, enum de role `ADMIN`/`OPERATOR` com default `OPERATOR`

### Evidência no Código
```typescript
// src/modules/auth/auth.service.ts:47-50
private signToken(userId: string, email: string, role: 'ADMIN' | 'OPERATOR'): string {
  const options: SignOptions = { expiresIn: env.JWT_EXPIRES_IN as SignOptions['expiresIn'] };
  return jwt.sign({ sub: userId, email, role }, env.JWT_SECRET, options);
}
```
```typescript
// src/middlewares/auth.middleware.ts:27-47
export const authenticate: RequestHandler = (req, _res, next) => {
  const header = req.headers.authorization;
  if (!header || !header.toLowerCase().startsWith('bearer ')) {
    next(new UnauthorizedError('Missing or invalid Authorization header'));
    return;
  }
  const token = header.slice(7).trim();
  // ...
  try {
    const payload = jwt.verify(token, env.JWT_SECRET) as JwtPayload;
    req.user = { id: payload.sub, email: payload.email, role: payload.role };
    next();
  } catch {
    next(new UnauthorizedError('Invalid or expired token'));
  }
};
```

### Análise de Impacto
- Introduzido: não determinável com granularidade — repositório iniciado em commit único (`7ef4317 init repository`, 2026-06-24 15:41:54 -0300); implementação já presente por completo desde a inicialização.
- Modificado: 0 commits incrementais registrados para `auth.service.ts`, `auth.middleware.ts` ou `env.ts` desde a inicialização (histórico de Git não informativo, consistente com o padrão já registrado nas demais análises desta Fase 2).
- Última alteração: 2026-06-24 (commit único de inicialização, sem tema específico além de "init repository").
- Afeta: 4 arquivos do módulo AUTH diretamente + 1 middleware compartilhado (`auth.middleware.ts`) + consumido por rotas de USERS, CUSTOMERS, PRODUCTS e ORDERS (todas usam `authenticate`/`requireRole`) — no mínimo 5 módulos, mais o futuro módulo WEBHOOKS.

### Alternativas (se observáveis)
Nenhuma alternativa explicitamente registrada em comentários de código, flags de configuração ou mensagens de commit (não há mecanismo alternativo de sessão/IdP presente ou comentado). Na transcrição, também não há alternativas de autenticação de usuário debatidas — o único debate de autenticação da reunião é sobre o mecanismo de webhooks (HMAC-SHA256), não sobre o JWT de usuários do OMS.

## Questões a abordar na ADR (se criada)
- Por que JWT stateless foi escolhido em vez de sessão com store (Redis/DB) ou um provedor de IdP externo (Auth0, Cognito, etc.)?
- A ausência de refresh token e de mecanismo de revogação/blacklist foi uma escolha deliberada de simplicidade para o escopo do projeto, ou uma lacuna a ser endereçada?
- O modelo de 2 papéis (`ADMIN`/`OPERATOR`) embutido diretamente no claim do JWT é suficiente para necessidades futuras, como o possível escopo por `customer_id` nos endpoints de configuração de webhook mencionados na reunião?
- Existe alguma política de rotação para `JWT_SECRET`? (Não há evidência disso no código, ao contrário do HMAC de webhooks, que decidiu por rotação com grace period de 24h — `[09:21]` Sofia.)
- Quais são as consequências de segurança aceitas de um `JWT_SECRET` vazado, dado que não há mecanismo de revogação de tokens já emitidos?

## ADRs Potenciais Relacionadas
- [`potential-adrs/must-document/WEBHOOKS/autenticacao-hmac-sha256-com-secret-por-endpoint.md`](../../must-document/WEBHOOKS/autenticacao-hmac-sha256-com-secret-por-endpoint.md) — mecanismo de autenticação **distinto** (HMAC-SHA256 com secret por endpoint, não JWT) para a integração outbound de webhooks com parceiros externos. Não deve ser confundido nem consolidado com esta ADR: a transcrição (`[09:20]-[09:22]` Sofia) trata a autenticação de webhooks como um mecanismo novo, propositalmente separado do JWT existente, para um escopo de confiança diferente (usuário interno autenticado da plataforma vs. parceiro externo recebendo callbacks HTTP). As duas ADRs, em conjunto, cobririam a "estratégia de autenticação" do sistema como um todo.

## Observações Adicionais
- **Lacuna sinalizada explicitamente**: `TRANSCRICAO.md` não contém nenhuma discussão direta sobre a escolha do JWT como mecanismo de autenticação de usuários do OMS, sobre bcrypt para hash de senha, ou sobre o modelo RBAC de 2 papéis — a reunião trata exclusivamente da feature de webhooks. JWT/RBAC são mencionados apenas de forma incidental, como infraestrutura já existente sendo reutilizada sem debate: em `[09:31]-[09:32]` (Marcos/Bruno/Larissa) ao esclarecer que o `customer_id` do cadastro de webhook não pode vir implícito do JWT (que pertence ao usuário operador interno, não ao cliente B2B) e precisa ser passado explicitamente no corpo/rota; e em `[09:35]-[09:36]` (Larissa/Sofia) ao decidir que o endpoint de replay de DLQ exige role `ADMIN`, reaproveitando o `requireRole` "que já existe". Portanto, esta análise se baseia **primariamente em evidência de código** (implementação robusta, validada por schema de env, usada por todos os módulos protegidos) e é elevada à Etapa 0 pela categoria de infraestrutura crítica de domínio "Autenticação" (sistema voltado a usuários) — não por uma decisão explicitamente confirmada em reunião. Isso é sinalizado aqui com honestidade: não é uma "decisão confirmada" no sentido de ter sido debatida/decidida nesta reunião específica, mas sim uma decisão de arquitetura pré-existente e estável, evidenciada pelo código e reforçada (não questionada) pela reunião.
- Diferente do HMAC de webhooks (rotação de secret com grace period de 24h, decidida em `[09:21]`), não há evidência no código de estratégia de rotação para `JWT_SECRET`. Isso não é uma incerteza da transcrição (o tema simplesmente não foi discutido, pois está fora do escopo da reunião), mas uma ausência observável na implementação atual — potencialmente relevante para uma seção de "riscos aceitos" na ADR formal.
- **Breakdown da pontuação**: Base Etapa 0 (categoria "Autenticação" — infraestrutura crítica de domínio para sistema voltado a usuários, com evidência de decisão estabelecida e estável no código) = 75; Escopo+Impacto = 20 (mais de 5 módulos: AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS, camada de enforcement em MIDDLEWARES, e o futuro módulo WEBHOOKS); Custo de Mudança = 20 (substituir o mecanismo por sessão com store ou IdP externo exigiria mudanças coordenadas em toda a API e nos clientes, estimado em 2-6 meses); Conhecimento da Equipe = 25 (todo engenheiro que cria um endpoint protegido precisa entender o mecanismo). **Total = 75 + 20 + 20 + 25 = 140/150.**
