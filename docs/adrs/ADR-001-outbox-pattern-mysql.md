# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status
Aceito

## Contexto
A nova feature de Webhooks de Notificação de Pedidos precisa disparar uma notificação HTTP para clientes B2B sempre que o status de um pedido muda. A mudança de status já ocorre dentro de uma transação Prisma em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que atualiza `orders`, insere em `order_status_history` e debita/reabastece `stockQuantity` dos produtos.

Na reunião, a equipe descartou disparo síncrono de HTTP dentro dessa transação: "Síncrono não rola. [...] Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos" [09:04 Bruno]; e "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá" [09:04 Bruno].

Era preciso um mecanismo que garanta consistência entre a mudança de status (fonte da verdade) e o registro do evento a ser notificado, sem acoplar a latência/disponibilidade de um endpoint externo à transação de negócio.

## Decisão
Adotar o padrão Outbox sobre o próprio MySQL já usado pela aplicação: "quando o status do pedido muda, dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox com o evento. [...] Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto" [09:06 Diego].

Um worker separado (ver [ADR-002](ADR-002-worker-processo-separado-polling.md)) lê a tabela `webhook_outbox` e realiza as chamadas HTTP de forma assíncrona. A inserção na outbox é feita por uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` client da transação em curso [09:41 Bruno/Diego], chamada a partir de `OrderService.changeStatus`.

A tabela tem índice composto em `status` + `created_at` para que o worker leia eficientemente apenas os pendentes mais antigos [09:07-08 Diego].

## Alternativas Consideradas
**Fila externa dedicada (ex. Redis Streams).** Consideradas e descartadas: "a alternativa seria botar Redis Streams ou alguma coisa parecida, mas a gente acabaria precisando subir mais infra. [...] a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" [09:07 Larissa/Diego]. O trade-off decisivo foi custo operacional de infraestrutura nova vs. reaproveitar o MySQL já existente, mesmo sabendo que isso implica polling em vez de push nativo.

## Consequências
**Positivas:**
- Consistência transacional garantida entre a mudança de status e o registro do evento — nenhuma inconsistência possível entre "status mudou" e "evento foi registrado" [09:06 Diego].
- Nenhuma infraestrutura nova: reutiliza o MySQL e o Prisma já em produção.
- Simplicidade operacional para um time pequeno.

**Negativas:**
- Introduz acoplamento de escrita: toda mudança de status agora grava também na outbox na mesma transação — se a inserção na outbox falhar, a transação inteira sofre rollback (comportamento desejado, mas aumenta a superfície de falha da transação de `changeStatus`).
- Sem notificação nativa (sem `LISTEN/NOTIFY` como em Postgres), a leitura depende de polling, introduzindo latência mínima (ver [ADR-002](ADR-002-worker-processo-separado-polling.md)).
- Arquivamento das linhas já entregues fica explicitamente fora do escopo desta feature [09:08 Diego], exigindo decisão futura sobre crescimento da tabela.
