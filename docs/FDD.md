# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica
Este documento detalha o "como construir" da feature cuja proposta está no [RFC](RFC.md) e cujas decisões isoladas estão nos [ADRs](adrs/). A aplicação é um Node.js + TypeScript/Express com Prisma sobre MySQL, organizada em módulos de domínio (`src/modules/{auth,users,customers,products,orders}`). A feature adiciona um novo módulo `src/modules/webhooks` e um novo entry point de processo, `src/worker.ts`, sem alterar o comportamento público dos módulos existentes além de um ponto de extensão em `order.service.ts`.

## Objetivos técnicos
- Publicar um evento de notificação, de forma transacionalmente consistente, sempre que `OrderService.changeStatus` muda o status de um pedido.
- Entregar esse evento a todos os webhooks cadastrados pelo customer que estejam inscritos naquele status, via HTTP assinado, dentro do SLA de latência (<10s, [09:02] Marcos) com alta probabilidade.
- Garantir que falhas de entrega não causem perda de eventos (retry + DLQ) nem bloqueiem a API principal.
- Expor CRUD de configuração de webhook e histórico de entregas para os clientes integradores, e um endpoint administrativo de reprocessamento de DLQ.

## Escopo e exclusões
Dentro do escopo: outbox, worker, retry/DLQ, HMAC, CRUD de webhook, histórico de entregas, replay de DLQ, integração em `order.service.ts`.

Fora de escopo (ver [PRD](PRD.md#escopo) para a visão de produto):
- Webhooks inbound (receber chamadas de clientes) [09:02-03].
- Notificação de fallback por e-mail em caso de falha repetida [09:37-38].
- Rate limiting de envio outbound — tratado como questão em aberto, não implementado nesta fase [09:38-39].
- Dashboard visual para o cliente acompanhar seus webhooks [09:39-40].
- Garantia de ordering global entre pedidos distintos [09:12-14].
- Arquivamento/limpeza de linhas entregues na outbox após 30 dias [09:08].

## Fluxos detalhados

### 1. Criação do evento na outbox
1. Um caller autenticado chama `PATCH /orders/:id/status` (endpoint já existente), que invoca `OrderService.changeStatus(id, input, userId)`.
2. Dentro da transação Prisma já existente em `changeStatus` (`src/modules/orders/order.service.ts`), após validar a transição (`canTransition`, `src/modules/orders/order.status.ts`) e aplicar débito/reposição de estoque, o service chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`.
3. `publishWebhookEvent`:
   - Busca os webhooks ativos do `customerId` do pedido cujo filtro de eventos inclui `toStatus` (filtro aplicado **na inserção**, não no envio — "se nenhum webhook do customer quer aquele status, nem insere. Economiza linha na tabela" [09:34 Bruno]).
   - Para cada webhook elegível, monta o payload completo (snapshot, ver [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)). Se o payload serializado exceder 64KB, **esse webhook específico é pulado** (não inserido, evento registrado em log como `WEBHOOK_PAYLOAD_TOO_LARGE`) sem afetar os demais webhooks elegíveis nem a transação de `changeStatus` — essa verificação é uma guarda defensiva síncrona em memória, anterior ao `INSERT`, e nunca lança uma exceção que propague para a transação SQL (ver "Matriz de erros" para o porquê desse código não ser HTTP-facing).
   - Para cada payload dentro do limite, insere uma linha em `webhook_outbox` com `id` (UUID), `webhookId`, `eventId` (UUID, usado depois como `X-Event-Id`), `payload` (JSON serializado), `status = 'PENDING'`, `attempts = 0`, `createdAt`.
4. Se a inserção em si falhar (erro de banco, não erro de tamanho de payload), a transação inteira sofre rollback — a mudança de status não é persistida sem o evento correspondente (garantia central do [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md)).

### 2. Processamento pelo worker
1. `src/worker.ts` inicializa um `PrismaClient` próprio (processo separado da API, [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)) e entra em loop de polling a cada 2 segundos.
2. A cada tick, busca um batch pequeno (ex. 20) das linhas `webhook_outbox` com `status = 'PENDING'`, ordenadas por `createdAt` ascendente, usando o índice composto em `(status, createdAt)`.
3. Para cada evento do batch, monta a requisição HTTP para a URL cadastrada no webhook, com os headers descritos na seção de Contratos Públicos, e envia com timeout de 10s.
4. **Sucesso (2xx):** marca a linha como `DELIVERED`, grava uma linha em `webhook_delivery` (histórico) com `statusCode`, `responseBody` (truncado), `durationMs`.
5. **Falha (timeout, erro de rede, status >= 400):** incrementa `attempts`, grava a tentativa em `webhook_delivery` com o erro, e agenda a próxima tentativa conforme a tabela de backoff (seção 3).

### 3. Retry
Backoff fixo por número de tentativa, não cumulativo ([ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md)):

| Tentativa | Intervalo após a falha anterior |
|---|---|
| 1 | imediata (primeira tentativa) |
| 2 | 1 minuto |
| 3 | 5 minutos |
| 4 | 30 minutos |
| 5 | 2 horas |
| 6 (última) | 12 horas |

A linha da outbox guarda `nextAttemptAt`; o worker só seleciona linhas `PENDING` com `nextAttemptAt <= now()`. Após a 6ª tentativa (5 retries) falhar, o evento é movido para DLQ (passo 4).

### 4. Dead Letter Queue (DLQ)
1. Ao esgotar as tentativas, o worker insere uma linha em `webhook_dead_letter` com `id` (UUID), `webhookId`, `eventId`, `payload`, `failureReason`, `lastStatusCode`, `createdAt`, e marca a linha original da outbox como `FAILED` (ou remove, mantendo só a DLQ como registro — decisão de implementação livre, desde que a outbox não acumule linhas mortas).
2. Um administrador pode reprocessar manualmente via `POST /admin/webhooks/dead-letter/:id/replay` (ver Contratos Públicos), que recria uma linha `PENDING` em `webhook_outbox` com `attempts = 0` a partir do payload da DLQ, e registra em log/auditoria qual usuário ADMIN executou o replay [09:35-36 Sofia].

## Contratos públicos

### Entrega do evento (chamada outbound do worker para o cliente)
Este não é um endpoint exposto pela nossa API — é a requisição HTTP que o **worker** faz para a `url` cadastrada no webhook, a cada evento entregue. É o contrato central da feature, referenciado pelos Fluxos (§2, passo 3).

- **Method:** `POST` para a `url` cadastrada no webhook.
- **Headers:**
  - `Content-Type: application/json` [09:44 Diego]
  - `X-Event-Id`: UUID do evento, gerado na inserção na outbox — usado pelo cliente para deduplicação (garantia at-least-once, ver [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)) [09:44 Diego]
  - `X-Signature`: HMAC-SHA256 do corpo da requisição, calculado com a secret do webhook (ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)) [09:44 Diego]
  - `X-Timestamp`: timestamp ISO 8601 do envio, para o cliente detectar replay attacks se quiser [09:44 Diego]
  - `X-Webhook-Id`: id do webhook cadastrado que disparou este envio, útil para um cliente com múltiplos webhooks saber qual cadastro caiu naquele envio [09:44-45 Sofia/Diego]
- **Body** (exemplo, campos definidos em [09:43 Diego]):
```json
{
  "event_id": "a1c9f3e2-4b7d-4e21-9c0a-1f2e3d4c5b6a",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-10T09:31:05.000Z",
  "order_id": "3c6a0f2e-7b1d-4e9a-8f3c-2d1e0a9b8c7d",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "SHIPPED",
  "customer_id": "7d2e1f4a-9c3b-4a5d-8e6f-1a2b3c4d5e6f",
  "total_cents": 15990
}
```
- **Resposta esperada do cliente:** qualquer status `2xx` é tratado como sucesso (marca `DELIVERED`); qualquer outro status ou timeout (10s) aciona retry (ver Fluxos §3).

**Endpoints de configuração e administração.** Todas as rotas abaixo, exceto a de replay, exigem apenas autenticação padrão (`authenticate`), sem middleware `requireRole` — "por enquanto sim. Mais pra frente a gente pode endurecer" [09:36-37 Sofia/Marcos]. Apenas a rota de replay de DLQ exige adicionalmente `requireRole('ADMIN')`.

### `POST /customers/:customerId/webhooks`
Cria um webhook para o customer. `customerId` vem do path, não do JWT (JWT é do usuário operador) [09:32 Bruno/Larissa].

Request:
```json
{
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```
Response `201 Created` (a secret só é retornada aqui, uma única vez):
```json
{
  "id": "0f1b7c3a-2e9d-4a6a-9c2a-7a0f4e8c9b11",
  "customerId": "3c6a0f2e-...",
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_7f3a9c...",
  "active": true,
  "createdAt": "2026-10-10T09:31:00.000Z"
}
```
Erros: `400 WEBHOOK_INVALID_URL` (URL não https), `400 VALIDATION_ERROR` (lista de `events` vazia ou inválida), `404 NOT_FOUND` (customer inexistente).

### `PATCH /webhooks/:id`
Atualiza `url`, `events` ou `active`.

Request:
```json
{ "events": ["SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`:
```json
{
  "id": "0f1b7c3a-2e9d-4a6a-9c2a-7a0f4e8c9b11",
  "customerId": "3c6a0f2e-...",
  "url": "https://atlas-comercial.example.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "updatedAt": "2026-10-10T10:00:00.000Z"
}
```
Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 VALIDATION_ERROR`.

### `POST /webhooks/:id/secret/rotate`
Gera uma nova secret para o webhook. A secret anterior permanece válida por 24h em paralelo, para o cliente migrar sem downtime [09:21-22 Sofia] (ver [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)).

Request: sem corpo — apenas `:id` do webhook no path.

Response `200 OK` (a nova secret só é retornada aqui, uma única vez):
```json
{
  "id": "0f1b7c3a-2e9d-4a6a-9c2a-7a0f4e8c9b11",
  "secret": "whsec_a92fd1...",
  "previousSecretValidUntil": "2026-10-11T09:31:00.000Z",
  "rotatedAt": "2026-10-10T09:31:00.000Z"
}
```
Durante a janela de `previousSecretValidUntil`, o worker aceita assinaturas verificáveis tanto com a secret nova quanto com a antiga (o cliente pode estar migrando); após esse timestamp, apenas a secret nova é aceita. Erros: `404 WEBHOOK_NOT_FOUND`.

### `DELETE /webhooks/:id`
Remove um webhook. Request: sem corpo — apenas `:id` do webhook no path. Response `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

### `GET /customers/:customerId/webhooks`
Lista webhooks do customer (paginado, reaproveitando `PaginatedResponse` de `src/shared/http/response.ts`).

Request (query string): `?page=1&pageSize=20`

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "0f1b7c3a-2e9d-4a6a-9c2a-7a0f4e8c9b11",
      "customerId": "3c6a0f2e-...",
      "url": "https://atlas-comercial.example.com/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-10T09:31:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

### `GET /webhooks/:id/deliveries`
Histórico dos últimos envios (até 100) [09:34 Marcos], paginado.

Request (query string): `?page=1&pageSize=20&status=FAILED`

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "7b2e...",
      "eventId": "a1c9...",
      "status": "DELIVERED",
      "statusCode": 200,
      "durationMs": 184,
      "attemptNumber": 1,
      "sentAt": "2026-10-10T09:31:05.000Z"
    },
    {
      "id": "7b2f...",
      "eventId": "a1ca...",
      "status": "FAILED",
      "statusCode": null,
      "error": "timeout após 10000ms",
      "attemptNumber": 1,
      "sentAt": "2026-10-10T09:40:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```
Erros: `404 WEBHOOK_NOT_FOUND`.

### `POST /admin/webhooks/dead-letter/:id/replay`
Requer `requireRole('ADMIN')`. Recoloca o item da DLQ como `PENDING` na outbox.

Request: sem corpo — apenas `:id` do item de DLQ no path.

Response `202 Accepted`:
```json
{ "outboxEventId": "9f3e2a1b-...", "status": "PENDING", "requeuedAt": "2026-10-10T11:00:00.000Z" }
```
Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN` (role insuficiente).

## Matriz de erros previstos (`WEBHOOK_*`)

| Código | HTTP Status | Classe (estende `AppError`) | Cenário |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` | `:id` de webhook inexistente em PATCH/DELETE/GET deliveries |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError` | URL cadastrada não é `https://` [09:23 Sofia] |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` | `:id` de item de DLQ inexistente no replay |
| `WEBHOOK_SECRET_REQUIRED` | — (interno, guarda defensiva) | — (não estende `AppError`; ver item 3 da Integração) | Tentativa de assinar/enviar uma entrega para um webhook sem secret ativa (estado de dados inconsistente; não deveria ocorrer, já que a secret é sempre gerada na criação) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — (não HTTP; não há requisição em curso nesse ponto — ver Fluxos §1) | — | Payload renderizado do evento excede 64KB no momento da montagem; **esse webhook específico é pulado** (log + não insere na outbox), sem afetar outros webhooks nem a transação de `changeStatus` [09:23-24 Sofia/Diego] |
| `WEBHOOK_DELIVERY_FAILED` | — (não HTTP; usado em log/`webhook_delivery.error`) | — | Falha de entrega HTTP pelo worker (timeout, 5xx do cliente, conexão recusada) |
| `WEBHOOK_EVENT_NOT_ELIGIBLE` | — (interno, não propagado como resposta HTTP) | — | Guarda defensiva: evento cujo `toStatus` não está na lista `events` do webhook (não deveria ocorrer dado o filtro na inserção; usado apenas em assertions/logs) |

> `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DELIVERY_FAILED` e `WEBHOOK_EVENT_NOT_ELIGIBLE` não são erros HTTP retornados a um chamador — são usados internamente (logs/guardas) porque ocorrem fora do ciclo de uma requisição síncrona (dentro da transação de `changeStatus` ou dentro do worker). Apenas `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_DEAD_LETTER_NOT_FOUND` são de fato retornados como resposta HTTP por um endpoint deste módulo.

Todas as classes `WEBHOOK_*` com status HTTP seguem o mesmo padrão de `src/shared/errors/http-errors.ts` (ex. `InsufficientStockError`, `InvalidStatusTransitionError`), sendo automaticamente tratadas pelo `error.middleware.ts` existente sem necessidade de alteração nele [09:29 Bruno].

## Estratégias de resiliência
- **Timeout:** 10 segundos por chamada HTTP do worker ao endpoint do cliente [09:42 Sofia/Diego].
- **Retry:** backoff fixo por tentativa (seção "Fluxos detalhados > 3. Retry"), 5 retries após a tentativa inicial.
- **Fallback:** Dead Letter Queue + replay manual via endpoint admin — não há fallback automático (ex. e-mail) nesta fase [09:37-38].
- **Isolamento de falha:** falha de entrega a um webhook nunca afeta a transação de negócio (`changeStatus` já committou antes do worker processar) nem a entrega a outros webhooks/clientes.
- **Limite de payload:** eventos com payload serializado acima de 64KB não são enviados; o webhook específico é pulado (log com `WEBHOOK_PAYLOAD_TOO_LARGE`) em vez de truncar o conteúdo ou abortar a transação de `changeStatus` [09:23-24 Sofia/Diego].

## Observabilidade
- **Métricas:** taxa de entrega com sucesso na primeira tentativa, taxa de eventos que caem em DLQ, idade (em segundos) do item mais antigo pendente em `webhook_outbox` (proxy de atraso do worker), contagem de replays manuais de DLQ por dia.
- **Logs:** via logger Pino já existente (`src/shared/logger/index.ts`), com `eventId`, `webhookId`, `attemptNumber` e `durationMs` em cada tentativa de entrega, seguindo o padrão de campos estruturados já usado pelo `error.middleware.ts` (`requestId`, `method`, `path`).
- **Tracing:** correlação ponta a ponta de um evento usando `X-Event-Id` como chave de busca em logs — da inserção na outbox (log no `order.service.ts`), passando por cada tentativa do worker, até a entrega final ou DLQ.

## Dependências e compatibilidade
- Mesmo MySQL e `DATABASE_URL` já usados pela API; quatro novas tabelas via migration Prisma — `webhook_endpoint`, `webhook_outbox`, `webhook_delivery`, `webhook_dead_letter` — mapeadas, respectivamente, pelos models Prisma `WebhookEndpoint`, `WebhookOutboxEvent`, `WebhookDelivery` e `WebhookDeadLetter` (ver seção "Integração" item 9), seguindo as convenções de `prisma/schema.prisma` (UUID como `@id @default(uuid())`, `@@map` para nome de tabela em snake_case, índices explícitos).
- Worker depende de um `PrismaClient` próprio por processo — não compartilha instância com a API [09:30 Bruno].
- Nenhuma nova dependência de infraestrutura (sem Redis, sem broker de mensagens) — decisão explícita no [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md).

## Critérios de aceite técnicos
- Uma mudança de status que gera evento elegível resulta em exatamente uma linha em `webhook_outbox` por webhook inscrito, inserida na mesma transação de `changeStatus`.
- Se a inserção na outbox falhar, a mudança de status inteira sofre rollback (teste de integração cobrindo esse caso).
- O worker entrega um evento pendente em até ~2s de latência adicional em condições normais (sem falha do cliente).
- Uma falha de entrega resulta em `attempts` incrementado e `nextAttemptAt` calculado conforme a tabela de backoff.
- Após a 6ª tentativa falhar (a inicial mais os 5 retries do backoff), o evento aparece em `webhook_dead_letter` e não é mais retentado automaticamente.
- `X-Signature` enviado é um HMAC-SHA256 válido calculável pelo cliente com a secret do seu endpoint.
- Secret antiga permanece válida por 24h após rotação.
- Replay de DLQ sem role ADMIN retorna `403 FORBIDDEN`.

## Riscos e mitigação
- **Acúmulo de linhas pendentes se o worker cair:** mitigado por o worker ser um processo monitorável independentemente da API (ver [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)) e pela métrica de "idade do item mais antigo pendente".
- **Crescimento não controlado da tabela `webhook_outbox`:** arquivamento fica fora de escopo desta feature [09:08], mas o índice em `(status, createdAt)` mantém o polling eficiente mesmo com crescimento, até que uma estratégia de arquivamento seja definida em fase futura.
- **Payload desatualizado por causa do snapshot:** mitigado intencionalmente — é o comportamento desejado (ver [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)); documentar claramente no portal do desenvolvedor que o payload reflete o estado no momento do evento, não o estado atual do pedido.

## Integração com o sistema existente
Esta seção detalha, arquivo por arquivo, como o módulo de webhooks se integra ao código já existente, sem alterar seu comportamento público:

1. **`src/modules/orders/order.service.ts`** — método `changeStatus` é estendido para, após validar a transição e aplicar o ajuste de estoque dentro da mesma transação Prisma, chamar `publishWebhookEvent(tx, order, from, to)`. Nenhuma assinatura pública do `OrderService` muda; a chamada é interna ao método. Se `publishWebhookEvent` lançar, o comportamento de rollback da transação (já existente via `this.prisma.$transaction`) cobre também o evento de webhook.
2. **`src/modules/orders/order.status.ts`** — nenhuma alteração de código, mas o módulo de webhooks depende de `toStatus` (já computado por `changeStatus` através de `canTransition`) para decidir, no momento da inserção, quais webhooks do customer estão inscritos naquele status.
3. **`src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts`** — as três classes de erro **HTTP-facing** do módulo (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookDeadLetterNotFoundError`) estendem `AppError` exatamente como `InsufficientStockError` e `InvalidStatusTransitionError` já fazem, reexportadas por um novo `src/modules/webhooks/webhook.errors.ts` (sem necessidade de modificar os arquivos existentes). Os códigos internos/guardas defensivas (`WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DELIVERY_FAILED`, `WEBHOOK_EVENT_NOT_ELIGIBLE`) **não** precisam estender `AppError` — nunca atravessam o `error.middleware.ts`, então são apenas identificadores usados em `logger.warn/error(...)` dentro do worker ou do service.
4. **`src/middlewares/auth.middleware.ts`** — o endpoint `POST /admin/webhooks/dead-letter/:id/replay` reutiliza diretamente `requireRole('ADMIN')`, já usado por outras rotas administrativas, sem necessidade de nova lógica de autorização.
5. **`src/middlewares/error.middleware.ts`** — nenhuma alteração necessária; como as novas classes de erro estendem `AppError`, o middleware já as trata corretamente (`err.statusCode`, `err.errorCode`, `err.details`).
6. **`src/shared/logger/index.ts`** — o worker (`src/worker.ts`) importa e usa a mesma instância/fábrica de logger Pino usada pela API, garantindo formato de log consistente (mesmos campos `service`, `env`, redaction de campos sensíveis).
7. **`src/server.ts`** — serve de modelo direto (bootstrap, shutdown gracioso em `SIGINT`/`SIGTERM`, conexão com `prisma`) para o novo `src/worker.ts`, que replica a mesma estrutura de bootstrap/shutdown, trocando o `app.listen` por um loop de polling.
8. **`src/routes/index.ts`** — o novo `buildWebhookRouter` (e o router administrativo de replay) é registrado aqui, seguindo exatamente o padrão de `router.use('/customers', buildCustomerRouter(...))` já usado para os demais módulos.
9. **`prisma/schema.prisma`** — quatro novos models (`WebhookEndpoint`, `WebhookOutboxEvent`, `WebhookDelivery`, `WebhookDeadLetter`) seguem a convenção já estabelecida: `id String @id @default(uuid()) @db.Char(36)`, `@@map` para nome de tabela em snake_case, índices explícitos nos campos de consulta frequente (análogo aos índices em `Order` e `Product`).
10. **`src/shared/http/response.ts`** — os endpoints `GET /customers/:customerId/webhooks` e `GET /webhooks/:id/deliveries` (ver "Contratos públicos") reaproveitam diretamente os tipos e a função `paginated()` já existentes neste arquivo (`Pagination`, `PaginatedResponse<T>`, `buildPagination`), sem necessidade de criar um novo padrão de paginação para o módulo de webhooks.
