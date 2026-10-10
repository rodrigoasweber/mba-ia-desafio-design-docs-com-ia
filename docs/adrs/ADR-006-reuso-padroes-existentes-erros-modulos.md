# ADR-006: Reuso dos padrões existentes de módulos, erros, logging e validação

## Status
Aceito

## Contexto
A aplicação já tem convenções consolidadas para organizar código de domínio: módulos em `src/modules/<dominio>` com `controller`/`service`/`repository`/`routes`/`schemas` (ver `src/modules/customers/`, `src/modules/orders/`, etc.), uma hierarquia de erros em `src/shared/errors/` (`AppError` e subclasses como `InsufficientStockError`, `InvalidStatusTransitionError`), um middleware de erro centralizado (`src/middlewares/error.middleware.ts`) e um logger Pino compartilhado (`src/shared/logger/index.ts`). A equipe discutiu explicitamente se o novo módulo de webhooks deveria seguir esses padrões ou introduzir algo novo.

## Decisão
O módulo de webhooks segue integralmente os padrões já estabelecidos no projeto:

- **Estrutura de módulo:** "a gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual. Vou propor uma pasta src/modules/webhooks com toda a estrutura" [09:27 Bruno], com a lógica de processamento do worker em um arquivo específico dentro do módulo (`webhook.worker.ts` ou `webhook.processor.ts`) [09:28 Bruno].
- **Erros:** códigos prefixados `WEBHOOK_*`, no mesmo espírito de `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`: "quero seguir igual pra webhook. Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc" [09:28 Bruno]. "Prefixo WEBHOOK_ pra tudo do módulo" [09:29 Larissa]. No detalhamento feito no FDD, os códigos que respondem a um chamador HTTP (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`) foram modelados como classes que estendem `AppError` (`src/shared/errors/app-error.ts`), exatamente como `InsufficientStockError`; os demais códigos `WEBHOOK_*` citados nesta reunião (como `WEBHOOK_SECRET_REQUIRED`) acabaram sendo guardas internas que nunca atravessam uma requisição HTTP e por isso não precisam estender `AppError` — ver [FDD](../FDD.md#matriz-de-erros-previstos-webhook_).
- **Logging e tratamento de erro:** reuso direto do logger Pino já presente no projeto inteiro e do middleware de erro centralizado existente (`src/middlewares/error.middleware.ts`), que já trata `AppError`, erros do Zod e do Prisma sem precisar de qualquer alteração: "o middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada" [09:29 Bruno].
- **Autorização:** reuso do middleware `requireRole` já existente em `src/middlewares/auth.middleware.ts` para proteger o endpoint de replay de DLQ com role `ADMIN` [09:36 Larissa].
- **Validação:** reuso do padrão de schemas Zod já usado em todos os módulos (ver `src/modules/customers/customer.schemas.ts`).

Resumo da decisão, confirmado pela tech lead: "reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros" [09:30 Larissa].

## Alternativas Consideradas
**Criar um subsistema de erros/validação próprio para o módulo de webhooks.** Não foi adotado — nenhum participante defendeu essa abordagem na reunião. O trade-off considerado implicitamente foi consistência e menor curva de aprendizado para quem já conhece a base de código vs. a possibilidade (descartada) de um módulo com particularidades técnicas (ex. assinatura HMAC, retry) justificar uma camada de erro/log diferenciada. A equipe optou por manter consistência total com o restante do projeto.

## Consequências
**Positivas:**
- Curva de aprendizado mínima para qualquer desenvolvedor já familiarizado com os módulos existentes (`orders`, `customers`, `products`).
- Nenhuma alteração necessária no middleware de erro, no logger ou na infraestrutura de autenticação/autorização — reduz risco de regressão em código compartilhado.
- Consistência de observabilidade: logs do módulo de webhooks aparecem no mesmo formato e pipeline dos demais módulos.

**Negativas:**
- Códigos de erro `WEBHOOK_*` HTTP (via `AppError`) não cobrem, por si só, erros específicos do fluxo assíncrono do worker (ex. falha de entrega HTTP para o cliente) — esses precisam de uma estratégia de log/observabilidade complementar, detalhada no [FDD](../FDD.md), já que não resultam em uma resposta HTTP para um chamador.
- Seguir o padrão existente à risca limita flexibilidade para otimizações específicas do domínio de webhooks (ex. schemas de erro mais ricos para contexto de entrega), caso surjam no futuro.
