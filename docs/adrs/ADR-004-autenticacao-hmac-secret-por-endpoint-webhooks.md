# ADR-004: Autenticação de Webhooks via HMAC-SHA256 com Secret Única por Endpoint e Rotação

**Status:** Proposta
**Date:** 2026-08-31

## Status

Proposta. A decisão foi fechada com consenso explícito da equipe na reunião de refinamento técnico,
sob condução da engenheira de segurança responsável, mas nenhuma parte do módulo de webhooks existe
hoje na base de código — não há geração, armazenamento ou verificação de secret, nem lógica de
assinatura HMAC implementada. O status reflete uma decisão de design aprovada e aguardando
implementação, com revisão de segurança dedicada já reservada antes do deploy.

## Contexto

O sistema de gestão de pedidos hoje expõe apenas autenticação interna via JWT stateless para
usuários operadores/administradores; não existe nenhum mecanismo de autenticação para consumidores
externos. Com a introdução de notificações de webhook para clientes B2B (Atlas, MaxDistribuição,
Nova Cargo), a plataforma passa a enviar dados de pedidos para infraestrutura fora de seu controle,
criando a necessidade de que cada cliente consiga validar que a requisição recebida realmente partiu
da plataforma e que o payload não foi adulterado em trânsito.

A equipe decidiu assinar cada evento com HMAC-SHA256, transmitindo a assinatura em um cabeçalho
dedicado. A motivação de negócio central para exigir uma secret única por endpoint cadastrado — em
vez de uma secret global da plataforma — foi um incidente real já vivido pela equipe, em que um
cliente vazou uma secret em log de sua própria aplicação; com secret por endpoint, um vazamento fica
isolado a um único cadastro, sem comprometer os demais clientes ou endpoints. A necessidade de
rotação da secret via API, com período de validade dupla (grace period) de 24 horas entre a secret
antiga e a nova, decorre da exigência prática de permitir que o cliente migre sistemas sem
indisponibilidade.

Como o módulo de webhooks ainda não existe, nem o esquema de dados do sistema
(`prisma/schema.prisma`) nem qualquer dependência de criptografia externa listada em `package.json`
refletem hoje esta necessidade — a única biblioteca criptográfica nativa disponível no projeto para
HMAC é o módulo `crypto` do Node, embora essa via não tenha sido especificada explicitamente pela
equipe durante a reunião. `[NEEDS INPUT: Não há definição, nem na transcrição nem no código, de como
a secret será armazenada em repouso no banco de dados — em texto plano, com hash, ou criptografada
com uma chave de aplicação/KMS — decisão que impacta diretamente o risco residual de um vazamento de
banco de dados.]`

## Decisão

A plataforma assinará o corpo de cada evento de webhook com HMAC-SHA256, transmitindo a assinatura
resultante em um cabeçalho de requisição dedicado. Cada endpoint de webhook cadastrado por cliente
terá sua própria secret, gerada e armazenada de forma independente das demais, permitindo rotação
individual sob demanda via API sem afetar outros cadastros. Durante a rotação, a secret antiga
permanece válida por 24 horas em paralelo à nova, evitando interrupção do lado do cliente enquanto
ele atualiza sua verificação.

A escolha de secret por endpoint em vez de secret global foi feita deliberadamente para reduzir o
raio de impacto ("blast radius") de um vazamento a um único cadastro, aceitando em troca maior
complexidade de armazenamento e gestão de ciclo de vida de múltiplas secrets — trade-off assumido
conscientemente pela equipe à luz de um incidente real já ocorrido com um cliente.

## Alternativas Consideradas

### Secret global compartilhada entre todos os endpoints/clientes

- **Prós:**
  - Modelo de armazenamento e distribuição mais simples (uma única secret por ambiente).
  - Menor esforço de implementação inicial no CRUD de configuração de webhook.
  - Elimina a necessidade de lógica de rotação por entidade individual.
- **Contras:**
  - Vazamento de uma única secret compromete a autenticidade de eventos para todos os clientes simultaneamente.
  - Foi exatamente o cenário de incidente já vivido pela equipe (vazamento em log de aplicação de um cliente).
  - Revogar a secret comprometida exige coordenar a atualização de todos os clientes ao mesmo tempo.
  - Não oferece isolamento de confiança entre clientes distintos, apesar de serem entidades de negócio independentes.

### Assinatura assimétrica (ex.: RSA ou ECDSA) em vez de HMAC simétrico

- **Prós:**
  - A chave pública poderia ser distribuída livremente ao cliente sem risco de comprometer a capacidade de assinatura da plataforma.
  - Elimina a necessidade de compartilhar um segredo simétrico com terceiros.
  - Reduz o impacto de um vazamento do lado do cliente, já que ele nunca possui a chave privada.
- **Contras:**
  - Maior complexidade de implementação e de gestão de par de chaves (geração, distribuição, expiração).
  - Não é o padrão predominante adotado por provedores de webhook de mercado, elevando o custo de integração para os clientes.
  - Overhead computacional maior por evento comparado a HMAC.
  - `[NEEDS INPUT: Não há evidência na transcrição de que assinatura assimétrica tenha sido de fato avaliada e descartada pela equipe; a escolha de HMAC-SHA256 foi apresentada e aceita diretamente como "padrão de mercado", sem registro de comparação formal com alternativas.]`

### Rotação com invalidação imediata da secret antiga (sem grace period)

- **Prós:**
  - Reduz a janela de tempo em que duas secrets distintas são aceitas simultaneamente para o mesmo endpoint.
  - Modelo de verificação mais simples do lado do cliente (uma única secret válida por vez).
  - Elimina a necessidade de lógica de expiração temporizada da secret antiga.
- **Contras:**
  - Exige que o cliente atualize sua secret exatamente no instante da rotação, sob risco de passar a rejeitar eventos legítimos.
  - Aumenta a chance de indisponibilidade percebida pelo cliente durante janelas de manutenção ou deploy do lado dele.
  - Contraria a exigência prática relatada pela equipe de permitir migração sem downtime do sistema do cliente.
  - `[NEEDS INPUT: Validar se a rotação sem grace period foi de fato avaliada e descartada pela equipe, ou se o grace period de 24h foi proposto diretamente sem comparação com uma alternativa de invalidação imediata.]`

## Consequências

A decisão garante autenticidade e integridade verificáveis para todo consumidor externo de eventos,
ao mesmo tempo em que isola o impacto de um vazamento de credencial a um único endpoint cadastrado,
em vez de comprometer toda a superfície de integração da plataforma. A rotação com grace period
elimina a necessidade de coordenação de downtime entre plataforma e cliente durante a troca de
secret, o que é especialmente relevante para os primeiros clientes B2B da plataforma, que passarão a
depender operacionalmente desses eventos.

Em contrapartida, a plataforma assume uma responsabilidade nova e permanente de gestão de ciclo de
vida de credenciais por entidade — geração, armazenamento seguro, rotação e expiração — problema que
o projeto nunca precisou resolver antes, já que a autenticação interna via JWT é stateless e não
depende de segredo por entidade. O grace period de 24 horas, embora necessário para migração sem
downtime, mantém deliberadamente duas secrets simultaneamente válidas por endpoint durante a janela
de rotação, o que amplia por um período limitado a superfície de credenciais aceitas. Por fim, a
criticidade da decisão já levou a equipe a reservar dias de revisão de segurança dedicada antes do
deploy, indicando um custo de processo recorrente associado a qualquer mudança futura neste
mecanismo.

Também não há, hoje, um procedimento definido para revogação de emergência de uma secret
comprometida antes do fim natural do grace period de 24 horas — cenário relevante caso um vazamento
seja identificado durante a própria janela de rotação. `[NEEDS INPUT: Falta definição de um fluxo de
revogação imediata (fora do grace period padrão) para o caso em que a equipe de segurança
identifique comprometimento ativo de uma secret durante a janela de dupla validade; nem a
transcrição nem o código endereçam esse cenário.]`

## Referências

- `TRANSCRICAO.md` — `[09:19]` a `[09:22]`, debate de decisão liderado por Sofia sobre HMAC-SHA256, secret por endpoint e rotação com grace period de 24h, incluindo a justificativa de incidente real trazida por Diego.
- `TRANSCRICAO.md` — `[09:46]`, reserva de revisão de segurança dedicada a "HMAC e geração de secret" antes do deploy, reforçando a criticidade operacional da decisão.
- `src/middlewares/auth.middleware.ts` — único mecanismo de autenticação existente no projeto (JWT stateless de usuário interno), contraste que confirma que a autenticação HMAC de webhooks é um mecanismo novo e não uma extensão do modelo atual.
- `prisma/schema.prisma` — ausência de qualquer campo de secret associado a endpoint de webhook, confirmando que o armazenamento de credenciais por endpoint é inteiramente novo.
- `src/shared/errors/app-error.ts` — padrão de hierarquia de erros da aplicação a ser reaproveitado para os futuros códigos `WEBHOOK_*` relacionados a falhas de assinatura e secret.
