# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status
Aceito

## Contexto
Dada a combinação de outbox + retry ([ADR-001](ADR-001-outbox-pattern-mysql.md), [ADR-003](ADR-003-retry-backoff-exponencial-dlq.md)), é possível que o mesmo evento seja entregue mais de uma vez ao cliente (por exemplo, se a entrega teve sucesso mas a confirmação/timeout do worker não foi processada a tempo). Era preciso decidir qual garantia de entrega oferecer e como o cliente poderia lidar com duplicatas.

## Decisão
O sistema garante **at-least-once delivery**: "pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" [09:24 Diego]. Para viabilizar deduplicação do lado do cliente, cada evento carrega um identificador único no header **`X-Event-Id`**, um UUID "gerado quando o evento entra na outbox. [...] único por evento. Se o cliente recebeu duas vezes, ele dedupica pelo event_id do lado dele" [09:25 Diego].

A responsabilidade de deduplicação é explicitamente transferida ao cliente: "isso joga responsabilidade pro cliente. [...] mas é o padrão de mercado. Stripe faz assim, GitHub faz assim" [09:25 Sofia/Diego]. Esse comportamento será documentado de forma destacada no portal de desenvolvedor para os clientes [09:26 Marcos].

## Alternativas Consideradas
**Garantia exactly-once.** Descartada: "garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" [09:25 Diego]. O trade-off foi complexidade de implementação e protocolo de coordenação bilateral vs. uma solução simples e amplamente adotada no mercado (precedentes citados: Stripe, GitHub), aceitando que a responsabilidade de dedup recai sobre o cliente.

## Consequências
**Positivas:**
- Implementação simples: nenhuma necessidade de coordenação transacional distribuída entre nosso sistema e o do cliente.
- Alinhado com padrão de mercado já conhecido por integradores (Stripe, GitHub), reduzindo custo de adoção.
- Compatível diretamente com o mecanismo de retry: reenviar um evento que já foi recebido não quebra a garantia, pois a responsabilidade de idempotência está no cliente.

**Negativas:**
- Transfere trabalho de implementação (dedup) para o lado do cliente — exige documentação clara e pode gerar fricção de integração para clientes que não implementarem a verificação de `X-Event-Id` corretamente.
- Não há garantia de entrega única nativa no nosso sistema; qualquer bug de dedup do lado do cliente pode gerar processamento duplicado de eventos de negócio por parte dele (risco fora do nosso controle direto, mitigado por documentação no portal do desenvolvedor).
