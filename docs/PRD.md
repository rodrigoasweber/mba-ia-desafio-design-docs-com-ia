# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature
O OMS não tem hoje nenhum mecanismo de notificação externa, eventos ou filas — esse vácuo é proposital e é exatamente o que esta feature endereça. Clientes B2B integrados à plataforma hoje descobrem mudanças de status de pedido fazendo polling repetido em `GET /orders`. Esta feature introduz um sistema de webhooks outbound: sempre que o status de um pedido muda, o sistema notifica, via HTTP assinado, os endpoints cadastrados pelos clientes interessados naquele status.

## Problema e motivação
Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — fizeram um pedido formal pedindo notificação em tempo real de mudança de status de pedido, pois o polling atual "tá deixando a integração lenta e cara pra eles" [09:00 Marcos]. A Atlas sinalizou risco de migração para um concorrente se a entrega não ocorrer até o fim do trimestre [09:00 Marcos]. Para esses clientes, "tempo real" significa um delay aceitável de até 10 segundos, não necessariamente notificação instantânea [09:02 Marcos].

## Público-alvo e cenários de uso
**Público-alvo:** clientes B2B integradores que consomem a API da plataforma (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), representados por usuários do sistema com JWT próprio (não um JWT "do cliente" — o cadastro de webhook é feito via API autenticada normalmente, com `customerId` informado explicitamente, não inferido do token) [09:32 Marcos/Larissa].

**Cenários de uso:**
1. Um integrador cadastra um endpoint de webhook para um customer, especificando a URL e os status de pedido de interesse (ex. apenas `SHIPPED` e `DELIVERED`).
2. Um pedido daquele customer muda de status; o integrador recebe, em seu endpoint, uma chamada HTTP assinada com os dados básicos da mudança, em até ~10 segundos.
3. O integrador consulta o histórico de entregas de um webhook específico para auditar sucessos/falhas recentes.
4. Um administrador da plataforma reprocessa manualmente um evento que caiu em Dead Letter Queue após falhas repetidas de entrega.

## Objetivos e métricas de sucesso
- **Latência de entrega:** p95 de entrega do evento de webhook em até 10 segundos a partir do commit da mudança de status, consistente com o requisito aceito pelos clientes ("abaixo de 10 segundos já é tempo real" [09:02 Marcos]) e com a decisão de polling do worker a cada 2s [09:09-10 Diego/Marcos].
- **Confiabilidade de entrega:** 100% dos eventos que esgotarem as tentativas automáticas de entrega (1 tentativa inicial + 5 retries) devem ficar visíveis e reprocessáveis na Dead Letter Queue — ou seja, nenhuma falha de entrega pode resultar em perda silenciosa de evento, apenas em um item auditável em DLQ (meta derivada diretamente da decisão de retry+DLQ, [09:17-19] Diego).
- **Eliminação de polling:** redução a zero das consultas de polling repetitivo a `GET /orders` pelos três clientes B2B iniciais, uma vez que a integração via webhook esteja ativa (objetivo qualitativo direto da motivação de negócio, [09:00] Marcos).

## Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por customer (CRUD de configuração).
- Filtro de eventos por status de pedido, por webhook.
- Entrega assíncrona via outbox + worker dedicado, com assinatura HMAC-SHA256 e garantia at-least-once.
- Retry com backoff exponencial e Dead Letter Queue para falhas persistentes.
- Endpoint administrativo de reprocessamento manual de itens em DLQ.
- Histórico de entregas por webhook (últimos 100 envios).

### Fora de escopo
- **Webhooks inbound** (o cliente enviar eventos para nós) — escopo é estritamente outbound, "eles querem receber, não mandar" [09:02-03 Marcos/Sofia].
- **Notificação de fallback por e-mail** quando um webhook falha repetidamente — levantado e explicitamente descartado para esta fase: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" [09:37-38 Larissa/Marcos].
- **Dashboard visual** para o cliente acompanhar seus webhooks — "não, agora não. Só endpoints. Painel é projeto separado do time de frontend" [09:39-40 Larissa].
- **Rate limiting de envio outbound** — levantado como preocupação, mas adiado: "a gente observa e implementa se virar problema" [09:38-39 Diego/Larissa].
- **Garantia de ordering global** entre pedidos distintos — documentada como limitação conhecida, não como requisito a ser construído [09:12-14].
- **Arquivamento de eventos entregues** na tabela outbox após 30 dias — explicitamente fora do escopo desta feature [09:08 Diego].

## Requisitos funcionais
1. Cliente (via API autenticada) cria um webhook informando `url`, lista de `events` (status de interesse) e `customerId`; o sistema gera e retorna a `secret` uma única vez na criação. O `customerId` é informado explicitamente no body/path, não inferido do JWT (o JWT é do usuário operador que opera a API, não "do cliente") [09:31 Marcos; correção de Bruno/Larissa em 09:32].
2. Cliente edita (`PATCH`) um webhook existente (url, events, status ativo) [09:33 Bruno].
3. Cliente remove (`DELETE`) um webhook [09:33 Bruno].
4. Cliente lista (`GET`) os webhooks cadastrados de um customer [09:33 Bruno].
5. Cada webhook filtra por lista de status de pedido que deseja receber; o filtro é aplicado na inserção do evento na outbox, não no momento do envio [09:33-34 Marcos/Bruno].
6. Cliente consulta o histórico de entregas de um webhook (`GET /webhooks/:id/deliveries`), com sucesso/falha, payload, resposta e tempo de resposta dos últimos 100 envios [09:34 Marcos].
7. Administrador reprocessa manualmente um evento em Dead Letter Queue via endpoint dedicado [09:18 Diego].
8. Apenas usuários com role `ADMIN` podem executar o reprocessamento de DLQ; a operação é registrada para auditoria [09:35-36 Sofia].
9. Demais endpoints de CRUD de configuração de webhook ficam acessíveis a qualquer usuário autenticado, independentemente de role, nesta fase [09:36-37 Sofia/Marcos].
10. A mudança de status de um pedido (`OrderService.changeStatus`) publica o evento de webhook dentro da mesma transação SQL que atualiza o pedido, garantindo consistência [09:40-41 Bruno/Diego].
11. O payload do evento contém `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents` — sem a lista de itens do pedido, para manter o payload enxuto [09:43 Diego/Bruno].
12. Cada requisição de entrega carrega os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` [09:44 Diego/Sofia].
13. O cliente pode solicitar rotação da secret de um webhook; a secret anterior permanece válida por 24 horas em paralelo [09:21-22 Sofia].
14. O identificador de evento (`event_id`) é um UUID gerado no momento da inserção na outbox, usado pelo cliente para deduplicação em cenários de entrega duplicada (at-least-once) [09:24-26 Diego].

## Requisitos não funcionais
- A URL cadastrada para um webhook deve ser obrigatoriamente `https://`; URLs `http://` são rejeitadas na validação [09:23 Sofia].
- O payload de um evento não pode exceder 64KB; eventos que excederiam esse limite falham explicitamente em vez de serem truncados [09:23-24 Sofia/Diego].
- O timeout de cada chamada HTTP de entrega é de 10 segundos [09:42 Sofia/Diego].
- A garantia de entrega é at-least-once (não exactly-once); duplicação de eventos é esperada e deve ser tratada pelo cliente [09:24-26 Diego].
- Assinatura de autenticidade via HMAC-SHA256 é obrigatória em toda requisição de entrega [09:20 Sofia].

## Decisões e trade-offs principais
As decisões arquiteturais centrais estão documentadas individualmente nos ADRs, cada um com o trade-off explícito que motivou a escolha:
- Outbox no MySQL em vez de fila dedicada — [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md)
- Worker em processo separado com polling de 2s — [ADR-002](adrs/ADR-002-worker-processo-separado-polling.md)
- Retry com backoff exponencial (5 retries após a tentativa inicial, 6 tentativas no total) e DLQ — [ADR-003](adrs/ADR-003-retry-backoff-exponencial-dlq.md)
- HMAC-SHA256 com secret por endpoint — [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- At-least-once com `X-Event-Id` — [ADR-005](adrs/ADR-005-at-least-once-x-event-id.md)
- Reuso de padrões de módulo/erro/log existentes — [ADR-006](adrs/ADR-006-reuso-padroes-existentes-erros-modulos.md)
- Snapshot do payload na inserção — [ADR-007](adrs/ADR-007-snapshot-payload-na-insercao.md)

A proposta técnica completa está no [RFC](RFC.md); o detalhamento de implementação está no [FDD](FDD.md).

## Dependências
- Banco MySQL e Prisma já existentes — nenhuma infraestrutura nova é introduzida (decisão explícita de evitar Redis/filas dedicadas, [ADR-001](adrs/ADR-001-outbox-pattern-mysql.md)).
- Middleware de autenticação/autorização já existente (`requireRole`) para proteger o endpoint de replay de DLQ.
- Disponibilidade da equipe de Segurança (Sofia) para revisão do código de HMAC e geração de secret antes do deploy — reservados no mínimo 2 dias úteis [09:46 Sofia].

## Riscos e mitigação
| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Endpoint do cliente indisponível por período prolongado (ex. manutenção planejada), esgotando as 6 tentativas de entrega (1 inicial + 5 retries) | Média — já ocorreu antes, caso real de indisponibilidade de 2h citado em reunião [09:16 Diego] | Médio — cliente não recebe a notificação automaticamente, mas pode consultar `GET /orders` como fallback | Dead Letter Queue + endpoint de replay manual para reprocessamento após o cliente voltar ao ar [09:17-19] |
| Vazamento da secret HMAC do lado do cliente | Média — incidente real já ocorrido com um cliente no passado [09:22 Diego] | Alto — permitiria forjar webhooks autenticados para aquele endpoint | Secret por endpoint (não global) e suporte a rotação com grace period de 24h [09:21-22 Sofia] |
| Acúmulo de eventos pendentes na outbox se o worker parar de processar | Baixa — worker roda como processo monitorável independente da API [09:11 Diego] | Médio — atraso na notificação, sem perda de dados (outbox é durável) | Processo de worker dedicado e monitorável, índice em `(status, created_at)` para leitura eficiente mesmo com acúmulo |

## Critérios de aceitação
- Um usuário autenticado consegue cadastrar, editar, listar e remover webhooks de um customer.
- Uma mudança de status de pedido gera, de forma transacionalmente consistente, um evento para cada webhook do customer inscrito naquele status.
- Um evento entregue com sucesso aparece no histórico de entregas do webhook correspondente.
- Um evento que falha em todas as 6 tentativas de entrega (1 inicial + 5 retries) aparece em Dead Letter Queue e pode ser reprocessado por um usuário ADMIN.
- Toda requisição de entrega contém uma assinatura HMAC-SHA256 válida e os headers `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id`.
- Uma URL de webhook não-HTTPS é rejeitada no cadastro/edição.

## Estratégia de testes e validação
- **Testes de integração** cobrindo o fluxo completo outbox → worker → entrega, com um endpoint HTTP mockado simulando sucesso, falha e timeout, seguindo o padrão já usado em `tests/orders.test.ts` (Vitest + Supertest).
- **Teste de consistência transacional:** forçar falha na inserção da outbox dentro de `changeStatus` e verificar que a mudança de status sofre rollback.
- **Teste de assinatura HMAC:** verificar que o `X-Signature` enviado é verificável com a secret correta e rejeitável com uma secret incorreta.
- **Teste de rotação de secret:** confirmar que a secret antiga continua válida durante o grace period de 24h e passa a ser inválida após esse período.
- **Teste de retry/DLQ:** simular 6 falhas consecutivas (a tentativa inicial + 5 retries) e verificar que o evento é movido para `webhook_dead_letter` e some da leitura ativa da outbox.
- **Teste de autorização:** confirmar que o endpoint de replay de DLQ retorna `403` para usuários sem role `ADMIN`.
