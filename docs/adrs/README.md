# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de Webhooks de Notificação de Pedidos, no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

Convenção de nomenclatura: `ADR-NNN-titulo-em-kebab-case.md`.

| ADR | Decisão |
|---|---|
| [ADR-001](ADR-001-outbox-pattern-mysql.md) | Padrão Outbox no MySQL |
| [ADR-002](ADR-002-worker-processo-separado-polling.md) | Worker em processo separado, polling 2s |
| [ADR-003](ADR-003-retry-backoff-exponencial-dlq.md) | Retry com backoff exponencial e Dead Letter Queue |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint |
| [ADR-005](ADR-005-at-least-once-x-event-id.md) | At-least-once com X-Event-Id |
| [ADR-006](ADR-006-reuso-padroes-existentes-erros-modulos.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-snapshot-payload-na-insercao.md) | Snapshot do payload na inserção da outbox |
