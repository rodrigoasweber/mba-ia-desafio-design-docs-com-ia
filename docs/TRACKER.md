# TRACKER de Rastreabilidade

Mapeia cada item relevante registrado no PRD, RFC, FDD e ADRs à sua origem na `TRANSCRICAO.md` (timestamp + falante) ou no código-fonte (caminho de arquivo). Itens cujo payload de conteúdo é puramente uma convenção de nomenclatura interna de baixo risco (ex. um único código de erro defensivo sem cenário de negócio associado) foram deliberadamente deixados fora da tabela em vez de receberem uma origem forçada — ver nota no [FDD](FDD.md#matriz-de-erros-previstos-webhook_), código `WEBHOOK_EVENT_NOT_ELIGIBLE`.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Requisito Não Funcional | p95 de entrega em até 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Decisão | 100% dos eventos esgotados ficam visíveis/reprocessáveis em DLQ (sem perda silenciosa) | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-03 | docs/PRD.md | Requisito Funcional | Eliminar polling repetitivo dos 3 clientes B2B | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Criar webhook (url, events, secret gerada) | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-01b | docs/PRD.md | Restrição | customerId vem do body/path, não do JWT (correção à proposta inicial de Marcos) | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar (PATCH) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover (DELETE) webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas (últimos 100) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay manual de evento em DLQ | TRANSCRICAO | [09:18] Diego |
| PRD-FR-08 | docs/PRD.md | Restrição | Replay de DLQ exige role ADMIN + auditoria | TRANSCRICAO | [09:35] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | CRUD de config aberto a qualquer autenticado | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Publicação do evento na mesma transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Campos do payload do evento | TRANSCRICAO | [09:43] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Headers de entrega (X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id) | TRANSCRICAO | [09:44] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | X-Event-Id (UUID) para dedup no cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | URL do webhook deve ser https | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de payload de 64KB, erro sem truncar | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once (não exactly-once) | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 obrigatória | TRANSCRICAO | [09:20] Sofia |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Webhooks inbound fora de escopo | TRANSCRICAO | [09:02-03] Marcos/Sofia |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Trade-off | Email de fallback adiado para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Trade-off | Dashboard visual fora de escopo | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Trade-off | Rate limiting outbound adiado | TRANSCRICAO | [09:38] Diego |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Sem garantia de ordering global | TRANSCRICAO | [09:13] Larissa |
| PRD-SCOPE-OUT-06 | docs/PRD.md | Restrição | Arquivamento de outbox fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-RISK-01 | docs/PRD.md | Risco | Endpoint do cliente indisponível por período prolongado | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Risco | Vazamento de secret HMAC do lado do cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Acúmulo de eventos pendentes se o worker cair | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia antes do deploy (≥2 dias úteis) | TRANSCRICAO | [09:46] Sofia |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Redis Streams descartado (overengineering p/ time pequeno) | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Exactly-once descartado (complexidade de coordenação) | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Decisão | Rate limiting outbound — observar e decidir depois | TRANSCRICAO | [09:38] Diego |
| RFC-OPEN-02 | docs/RFC.md | Decisão | Fallback de e-mail — adiado | TRANSCRICAO | [09:37] Larissa |
| RFC-OPEN-03 | docs/RFC.md | Decisão | Escala do worker para múltiplas instâncias — não decidido | TRANSCRICAO | [09:13] Diego |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-mysql.md | Decisão | Outbox pattern no MySQL | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-processo-separado-polling.md | Decisão | Worker em processo separado, polling 2s | TRANSCRICAO | [09:08] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-exponencial-dlq.md | Decisão | Retry com backoff exponencial e DLQ | TRANSCRICAO | [09:15] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-005-at-least-once-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes-erros-modulos.md | Decisão | Reuso de padrões de módulo/erro/log existentes | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CODE | docs/adrs/ADR-006-reuso-padroes-existentes-erros-modulos.md | Decisão | Base de erros reutilizada (AppError) | CODIGO | src/shared/errors/app-error.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Decisão | Snapshot do payload na inserção da outbox | TRANSCRICAO | [09:51-52] Bruno/Larissa |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /customers/:customerId/webhooks | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | GET /customers/:customerId/webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | POST /webhooks/:id/secret/rotate | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Requisito Funcional | Contrato outbound de entrega (headers X-Event-Id/X-Signature/X-Timestamp/X-Webhook-Id) | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | docs/FDD.md | Decisão | Código WEBHOOK_NOT_FOUND | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Decisão | Código WEBHOOK_INVALID_URL | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Decisão | Código WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Decisão | Código WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-05 | docs/FDD.md | Decisão | Código WEBHOOK_DEAD_LETTER_NOT_FOUND | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-06 | docs/FDD.md | Decisão | Código WEBHOOK_DELIVERY_FAILED (log, não HTTP) | TRANSCRICAO | [09:15] Diego |
| FDD-FLUXO-01 | docs/FDD.md | Requisito Funcional | Batch pequeno de leitura por tick do worker | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Tabela de progressão de backoff (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | DLQ em tabela separada webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Restrição | Worker usa PrismaClient próprio por processo | TRANSCRICAO | [09:30] Bruno |
| FDD-FLUXO-05 | docs/FDD.md | Decisão | IDs de outbox/webhook/DLQ em UUID, seguindo convenção do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-RESILIENCIA-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RESILIENCIA-02 | docs/FDD.md | Decisão | Sem fallback automático (ex. e-mail) nesta fase | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-01 | docs/FDD.md | Decisão | Reuso do logger Pino no worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEGRACAO-01 | docs/FDD.md | Decisão | Extensão de OrderService.changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Decisão | Dependência de toStatus/canTransition | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Decisão | Novas classes de erro estendendo AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Decisão | Reuso de padrão de erros HTTP existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGRACAO-05 | docs/FDD.md | Decisão | Reuso de requireRole para endpoint admin | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRACAO-06 | docs/FDD.md | Decisão | Nenhuma alteração necessária no middleware de erro | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRACAO-07 | docs/FDD.md | Decisão | src/worker.ts espelha bootstrap de src/server.ts | CODIGO | src/server.ts |
| FDD-INTEGRACAO-08 | docs/FDD.md | Decisão | Registro do router de webhooks | CODIGO | src/routes/index.ts |
| FDD-INTEGRACAO-09 | docs/FDD.md | Decisão | Novos models seguindo convenção de schema.prisma | CODIGO | prisma/schema.prisma |
| FDD-INTEGRACAO-10 | docs/FDD.md | Decisão | Paginação de listagem reaproveitada | CODIGO | src/shared/http/response.ts |

## Cobertura
- Total de linhas: 78.
- Linhas com Fonte = TRANSCRICAO: 66 (≈85%), todas com timestamp no formato `[hh:mm] Nome`.
- Linhas com Fonte = CODIGO: 12 (≥5 exigido), todas com caminho de arquivo real, confirmado via leitura direta do repositório base.
- Itens identificáveis cobertos: todos os requisitos funcionais e não funcionais do PRD, todas as alternativas e questões em aberto do RFC, todos os 7 ADRs, os 7 endpoints de configuração/administração mais o contrato de entrega outbound, e a maior parte da matriz de erros e da seção de integração do FDD — acima do mínimo de 80% exigido.
