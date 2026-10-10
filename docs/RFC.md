# RFC: Sistema de Webhooks de Notificação de Pedidos

## Metadados
- **Autor:** Rodrigo (via produção assistida por IA, a partir da reunião técnica)
- **Status:** Em revisão
- **Data:** 2026-10-10
- **Revisores:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, time de Pedidos), Diego (Eng. Sênior, time de Plataforma), Sofia (Engenharia de Segurança)

## Resumo executivo (TL;DR)
Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pediram notificação em tempo real de mudanças de status de pedido, hoje obtida via polling repetido em `GET /orders` [09:00 Marcos]. Propomos um sistema de webhooks outbound: a mudança de status em `OrderService.changeStatus` grava um evento em uma tabela outbox (MySQL) na mesma transação; um worker dedicado, em processo separado, faz polling dessa tabela e entrega os eventos via HTTP assinado (HMAC-SHA256), com retry exponencial e Dead Letter Queue (DLQ) para falhas persistentes. O documento detalha a proposta, as alternativas descartadas e os pontos ainda em aberto; o "como construir" fica detalhado no [FDD](FDD.md).

## Contexto e problema
A aplicação não tem hoje nenhum mecanismo de notificação externa, eventos ou filas. Os três clientes B2B citados ficam "batendo no GET /orders de tempos em tempos pra ver se mudou alguma coisa, e isso tá deixando a integração lenta e cara pra eles" [09:00 Marcos]. A Atlas sinalizou possível migração para concorrente se a feature não for entregue até o fim do trimestre [09:00 Marcos]. O requisito de latência aceito pelos clientes é "qualquer coisa abaixo de 10 segundos já é 'tempo real'" [09:02 Marcos]. O escopo é estritamente outbound (nós → cliente); os clientes não precisam enviar webhooks para nós [09:02-03 Sofia/Marcos].

A restrição central do problema é arquitetural: a mudança de status já ocorre dentro de uma transação pesada (`order.service.ts::changeStatus`, que atualiza `orders`, insere em `order_status_history` e ajusta `stockQuantity`). Qualquer solução não pode acoplar a disponibilidade/latência de um endpoint HTTP externo a essa transação — "se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos" [09:04 Bruno].

## Proposta técnica
Em visão geral (detalhamento de contratos e fluxos no [FDD](FDD.md)):

1. **Captura do evento:** dentro da mesma transação de `changeStatus`, uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` insere um evento na tabela `webhook_outbox`, já filtrado pelos status que cada webhook do cliente deseja receber, e com o payload já renderizado (snapshot) — ver [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md) e [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md).
2. **Entrega assíncrona:** um worker dedicado (`src/worker.ts`, processo separado da API) faz polling da outbox a cada 2 segundos e envia os eventos pendentes via HTTP, com timeout de 10s — ver [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md).
3. **Autenticidade:** cada requisição é assinada com HMAC-SHA256 usando uma secret própria por endpoint de webhook cadastrado, rotacionável com grace period de 24h — ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md).
4. **Garantia de entrega:** at-least-once, com deduplicação do lado do cliente via header `X-Event-Id` — ver [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md).
5. **Resiliência:** falhas de entrega acionam retry com backoff exponencial (5 retries após a tentativa inicial, 6 tentativas no total); após esgotadas, o evento vai para uma Dead Letter Queue reprocessável manualmente por um endpoint admin — ver [ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md).
6. **Padrões de implementação:** o módulo segue a estrutura e as convenções já existentes no projeto (módulos, `AppError`, logger, middlewares de auth/erro) — ver [ADR-006](adrs/ADR-006-reuso-padroes-existentes-erros-modulos.md).

Essa composição resolve o problema sem introduzir nenhuma peça de infraestrutura nova: tudo roda sobre o MySQL e o Node.js já em produção.

## Alternativas consideradas
**Fila de mensagens dedicada (ex. Redis Streams) em vez de outbox no MySQL.** Descartada porque exigiria subir e operar infraestrutura nova para um time pequeno: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" [09:07 Diego]. Trade-off: uma fila dedicada ofereceria push nativo (sem polling), mas ao custo de complexidade operacional desproporcional ao problema atual.

**Garantia de entrega exactly-once.** Descartada em favor de at-least-once: "garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" [09:25 Diego]. Trade-off: exactly-once eliminaria a necessidade de dedup no cliente, mas exigiria um protocolo de coordenação bilateral incompatível com o prazo e a simplicidade desejada; o padrão de mercado (Stripe, GitHub) usa a mesma abordagem adotada aqui.

## Questões em aberto
1. **Rate limiting de envio outbound.** Se um cliente tiver muitos pedidos mudando de status em um curto intervalo, hoje não há limite de taxa de chamadas para o endpoint dele. A equipe decidiu observar o comportamento em produção antes de decidir se é necessário: "eu acho que não [faz parte do escopo]. A gente observa e implementa se virar problema. Mas vale registrar como ponto em aberto" [09:38-39 Diego/Larissa].
2. **Notificação de fallback (ex. e-mail) quando um webhook falha repetidamente.** Foi levantado e explicitamente adiado: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" [09:37-38 Larissa].
3. **Estratégia de escala do worker para múltiplas instâncias.** Hoje o design assume single-worker para preservar ordering por `order_id`. Se for necessário escalar, "dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora" [09:13 Diego] — nenhuma estratégia foi decidida.

## Impacto e riscos
- Indisponibilidade prolongada de um endpoint de cliente esgota as 6 tentativas de entrega (1 inicial + 5 retries) e move o evento para DLQ, exigindo replay manual (ver [ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md)); mitigação detalhada no [PRD](PRD.md#riscos-e-mitigação).
- Vazamento de secret HMAC do lado do cliente é um risco real e já ocorrido no passado [09:22 Diego]; mitigado por secret por endpoint e rotação com grace period (ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).
- Nenhum impacto em disponibilidade da API principal: a integração no `order.service.ts` adiciona apenas uma escrita na mesma transação já existente, sem chamadas de rede síncronas.

## Decisões relacionadas
- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-mysql.md)
- [ADR-002 — Worker em processo separado, polling 2s](adrs/ADR-002-worker-processo-separado-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-exponencial-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-x-event-id.md)
- [ADR-006 — Reuso de padrões existentes](adrs/ADR-006-reuso-padroes-existentes-erros-modulos.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-payload-na-insercao.md)
