# ADR-002: Worker dedicado em processo separado, com polling de 2 segundos

## Status
Aceito

## Contexto
Com o padrão Outbox decidido ([ADR-001](ADR-001-outbox-pattern-mysql.md)), era preciso definir como os eventos pendentes em `webhook_outbox` seriam lidos e entregues. A equipe também precisava decidir onde esse processamento roda em relação à API HTTP existente (`src/server.ts`).

A exigência de negócio para latência é "abaixo de 10 segundos já é 'tempo real'" [09:02 Marcos], o que define o teto aceitável para o mecanismo de leitura escolhido.

## Decisão
O worker lê a outbox por **polling em loop, a cada 2 segundos**: "a cada 2 segundos, busca os eventos pendentes mais antigos, processa, marca" [09:09 Diego]. Esse intervalo atende com folga o requisito de <10s [09:09-10 Diego/Marcos].

O worker roda como **processo separado da API**, não dentro da mesma instância: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker" [09:11 Diego]. Isso é implementado como um novo entry point `src/worker.ts`, espelhando o padrão de bootstrap já usado em `src/server.ts`, com script dedicado `npm run worker` [09:11 Larissa]. O worker conecta no mesmo banco (mesma `DATABASE_URL`), mas com uma instância própria de `PrismaClient`, já que "PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova porque é outro processo Node" [09:30 Bruno].

## Alternativas Consideradas
**Trigger de banco de dados.** Descartada porque "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito" [09:09 Diego]. O trade-off: reatividade imediata exigiria um mecanismo improvisado e frágil fora do que o MySQL oferece nativamente; polling de 2s é suficiente para o SLA de 10s sem essa complexidade.

**Worker na mesma instância/processo da API.** Implicitamente descartada: perderia o evento em processamento a cada restart/deploy da API [09:11 Diego].

## Consequências
**Positivas:**
- Resiliência a restarts da API: o worker continua operando independente de deploys/reinícios do servidor HTTP.
- Implementação simples, sem dependência de infraestrutura de mensageria.
- Latência previsível e dentro do SLA (pior caso ~2s de espera antes do processamento, bem abaixo de 10s).

**Negativas:**
- **Limitação de ordering conhecida:** com um único worker, a ordem de entrega respeita `created_at`, garantindo ordem apenas por `order_id`; não há garantia de ordering global entre pedidos diferentes, e essa garantia se perde caso o worker seja escalado para múltiplas instâncias no futuro [09:12-13 Diego]. Documentado como limitação aceita, não como requisito ("os clientes nunca pediram garantia de ordering global, eles só querem saber se cada pedido deles mudou" [09:14 Marcos]).
- Overhead constante de polling no banco mesmo quando não há eventos pendentes (custo aceito, mitigado pelo índice em `status`/`created_at`).
- Escalar para múltiplos workers no futuro exigirá particionamento por `order_id` ou lock pessimista — explicitamente tratado como "problema do futuro, não agora" [09:13 Diego], e listado como questão em aberto no [RFC](../RFC.md).
