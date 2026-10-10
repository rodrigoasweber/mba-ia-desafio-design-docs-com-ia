# ADR-007: Payload do evento renderizado (snapshot) no momento da inserção na outbox

## Status
Aceito

## Contexto
Ao modelar a tabela `webhook_outbox`, surgiu a dúvida de quando o payload JSON enviado ao cliente deveria ser montado: no momento em que o evento é inserido na outbox (dentro da transação de `changeStatus`), guardando o payload já pronto, ou guardando apenas uma referência (`order_id`) e montando o payload somente no momento do envio pelo worker.

## Decisão
O payload é **renderizado e persistido no momento da inserção** na outbox, como um snapshot do estado do pedido naquele instante: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio? [...] Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" [09:51-52 Bruno/Larissa]. Diego e Bruno concordaram explicitamente com a abordagem de snapshot [09:52].

Na prática, isso significa que a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` (ver [ADR-001](ADR-001-outbox-pattern-mysql.md)) monta o JSON completo do evento (conforme contrato descrito no [FDD](../FDD.md)) e grava esse JSON já serializado na coluna de payload da `webhook_outbox`, dentro da mesma transação da mudança de status.

## Alternativas Consideradas
**Guardar apenas `order_id` e renderizar o payload no momento do envio.** Descartada porque geraria inconsistência se o pedido fosse alterado entre o momento da mudança de status e o momento em que o worker efetivamente envia o evento (que pode ser adiado por retries): o cliente receberia um payload refletindo o estado *atual* do pedido, não o estado no momento exato da transição de status que originou a notificação [09:51-52 Larissa]. O trade-off foi espaço de armazenamento extra (payload duplicado por evento) vs. correção semântica do evento histórico.

## Consequências
**Positivas:**
- O evento entregue ao cliente sempre reflete fielmente o estado do pedido no momento exato da transição de status, mesmo que o pedido seja modificado depois ou que a entrega seja adiada por retries.
- O worker não precisa fazer nenhuma consulta adicional ao pedido no momento do envio — apenas lê e envia o payload já armazenado, simplificando o processamento e reduzindo carga de leitura no banco durante o envio.

**Negativas:**
- Maior consumo de espaço em disco na tabela `webhook_outbox`, já que cada linha carrega o payload completo (ainda que o payload seja intencionalmente enxuto — sem `items`, conforme decisão de formato no [FDD](../FDD.md)).
- Qualquer alteração futura no formato do payload exige cuidado com eventos antigos já persistidos no formato anterior (compatibilidade de schema na leitura pelo worker), caso o `webhook_outbox` acumule eventos de versões diferentes do payload ao longo do tempo.
