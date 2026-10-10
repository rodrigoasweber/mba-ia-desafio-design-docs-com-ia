# ADR-003: Retry com backoff exponencial (5 retries) e Dead Letter Queue

## Status
Aceito

## Contexto
Clientes B2B podem estar temporariamente indisponíveis ou lentos quando o worker tenta entregar um evento. A equipe discutiu como lidar com falhas de entrega sem deixar eventos "pendurados" indefinidamente, nem desistir cedo demais de clientes com indisponibilidade pontual — o time já teve "cliente nosso com indisponibilidade de duas horas em manutenção planejada" [09:16 Diego].

## Decisão
Em caso de falha na entrega (timeout, erro HTTP, conexão recusada), o worker reagenda a tentativa com **backoff exponencial**, usando a progressão **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** entre uma falha e a tentativa seguinte [09:17 Diego]. Isso são **5 retentativas após o envio inicial — 6 tentativas no total** — cobrindo uma janela de quase 15 horas entre a primeira falha e a última tentativa: "Eu pensei em 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas. Total de quase 15 horas entre primeira falha e última tentativa" [09:17 Diego].

> **Nota de precisão:** quando perguntado "quantas tentativas?" [09:15 Bruno], Diego responde "Eu sugiro 5" [09:15 Diego], o que poderia ser lido como 5 tentativas totais. Mas a progressão de backoff que ele mesmo detalha em seguida [09:17] lista exatamente 5 intervalos de espera, e a soma desses 5 intervalos (1+5+30+120+720 min = 876 min ≈ 14,6h) é a única leitura compatível com o "quase 15 horas" que ele próprio afirma. Isso só fecha matematicamente se os "5" forem 5 *retentativas* após uma tentativa inicial imediata — 6 tentativas no total. Adotamos essa leitura (6 tentativas totais) de forma consistente em todo o pacote de documentos, para não haver ambiguidade entre ADR, FDD e PRD.

Após a **6ª tentativa** (a inicial mais os 5 retries) falhar, o evento é movido para uma tabela separada, **`webhook_dead_letter`**, contendo o payload, o motivo da falha e o timestamp: "eu fazia uma tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp. Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" [09:18 Diego].

O reprocessamento de itens em DLQ é **manual**, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que "recoloca na outbox como pendente" [09:18 Diego]. Esse endpoint exige role `ADMIN` e deve logar quem executou o replay para auditoria [09:35-36 Sofia/Larissa] (ver [ADR-006](ADR-006-reuso-padroes-existentes-erros-modulos.md) para o reuso do middleware `requireRole`).

## Alternativas Consideradas
**Retry indefinido com backoff.** Descartado: "algumas pessoas defendem retry indefinido com backoff, mas isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" [09:15 Diego]. Trade-off: simplicidade de não precisar de DLQ vs. risco de acumular eventos órfãos indefinidamente.

**3 tentativas.** Descartado por ser agressivo demais: "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria" [09:16 Diego]. Trade-off: menor custo de reprocessamento vs. maior taxa de falsos negativos (eventos descartados por indisponibilidade temporária e curta do cliente).

## Consequências
**Positivas:**
- Cobre janelas realistas de indisponibilidade de cliente (até ~15h) sem intervenção manual.
- Nenhum evento é perdido: o que não é entregue automaticamente fica em DLQ, auditável e reprocessável.
- Separação clara entre outbox "operacional" (pendentes/recentes) e DLQ "exceções", mantendo a leitura do worker rápida.

**Negativas:**
- Eventos em DLQ não são reentregues automaticamente — exige operação manual via endpoint admin, o que pode gerar atraso adicional de notificação se ninguém monitorar a DLQ ativamente (mitigação de monitoramento fica como responsabilidade de observabilidade, ver [FDD](../FDD.md)).
- Janela de 15h de retry mantém linhas "em voo" na outbox por mais tempo, aumentando o volume médio da tabela.
- Rate limiting de envio em cenários de muitas falhas simultâneas não foi definido nesta decisão — ficou como questão em aberto [09:38-39 Diego/Larissa].
