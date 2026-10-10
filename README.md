# Da Reunião ao Documento — Sistema de Webhooks de Notificação de Pedidos

Pacote de design docs (PRD, RFC, FDD, ADRs e Tracker de rastreabilidade) produzido a partir da `TRANSCRICAO.md` da reunião técnica e do código-fonte real deste repositório, para a feature de Webhooks de Notificação de Pedidos do OMS.

## Sobre o desafio
O desafio pede para transformar a transcrição de uma reunião técnica (cinco participantes, ~55 minutos) em um pacote completo de documentação de engenharia — PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade — suficiente para um time começar a implementar a feature sem precisar voltar à gravação da call. A restrição central é que nenhuma informação pode ser inventada: todo requisito, decisão ou restrição registrado precisa ser rastreável a um timestamp da transcrição ou a um arquivo real do código-fonte da aplicação (um OMS em Node.js/TypeScript/Prisma/MySQL já existente, sem nenhum mecanismo de notificação externa).

O trabalho não é sobre escrever prosa bonita — é sobre extrair corretamente o que foi decidido, o que foi descartado e o que ficou em aberto na reunião, e distribuir essa informação na "altura" certa: PRD (produto), RFC (arquitetura, para revisão), ADRs (decisões pontuais), FDD (implementação) e Tracker (rastreabilidade transversal), sem repetir conteúdo entre eles.

## Ferramentas de IA utilizadas
- **Claude Code (Claude Sonnet 5)** — ferramenta única de produção de todo o pacote: leitura do repositório base real via `gh api` (sem clonar, na fase de planejamento), leitura integral da `TRANSCRICAO.md`, extração estruturada de decisões/requisitos/itens descartados, e geração de todos os 11 arquivos Markdown (PRD, RFC, FDD, 7 ADRs, Tracker) e deste README. Também usada para o setup do repositório (`gh repo fork` + `git clone`).

## Workflow adotado
1. **Planejamento antes de gerar qualquer texto:** entrei em modo de planejamento e usei `gh api` para ler, direto do repositório base no GitHub, os arquivos de código realmente relevantes (`order.service.ts`, `order.status.ts`, `app-error.ts`, `http-errors.ts`, middlewares de auth/erro, logger, `server.ts`, `routes/index.ts`, `schema.prisma`) e a `TRANSCRICAO.md` completa, **antes** de escrever qualquer documento. Isso evitou citar caminhos de arquivo ou nomes de função que não existem no projeto.
2. **Inventário único de decisões/requisitos/descartes**, extraído da transcrição com prompts dirigidos (não genéricos) e organizado por timestamp — essa lista foi a fonte de verdade reaproveitada em todos os documentos seguintes, para evitar inconsistência entre eles.
3. **ADRs primeiro** (7 arquivos), um por decisão isolada identificada na reunião, cobrindo as 6 decisões obrigatórias do desafio mais 1 decisão secundária (snapshot do payload na inserção do evento).
4. **RFC**, consolidando os ADRs em uma proposta única de arquitetura, com alternativas descartadas e questões em aberto.
5. **FDD**, detalhando fluxos, contratos HTTP, matriz de erros `WEBHOOK_*` e a seção obrigatória de integração, citando 12 arquivos reais do código base em 10 pontos de integração.
6. **PRD**, escrito por último entre os documentos grandes, como consolidação de produto em cima do que já estava decidido em RFC/FDD/ADRs.
7. **Tracker**, construído varrendo os documentos já prontos e extraindo um identificador por item relevante, com origem em timestamp ou caminho de arquivo.
8. **Este README**, escrito ao final, documentando o processo já concluído.

## Prompts customizados
Dois exemplos representativos usados durante a produção (adaptados aqui para leitura; a intenção foi preservada):

**1 — Extração dirigida de decisões e descartes (usado para montar o inventário antes de qualquer documento):**
```
Leia a TRANSCRICAO.md completa. Para cada decisão arquitetural fechada,
extraia: o que foi decidido, quem decidiu, o timestamp [hh:mm] e o nome
do falante, e qual alternativa concreta foi colocada na mesa e descartada
(com o trade-off explícito dado em reunião, não inventado).

Separe em uma segunda lista tudo que foi EXPLICITAMENTE descartado ou
adiado para fase futura — não inclua como requisito em nenhum documento
posterior. Marque cada item descartado com o timestamp exato em que isso
foi dito. Não infira decisões que não foram verbalizadas.
```

**2 — Geração de um ADR a partir de uma decisão isolada do inventário, sem acesso ao resto da transcrição:**
```
Com base apenas nestes trechos da transcrição (colados abaixo, com
timestamp), escreva um ADR em formato MADR para a decisão de usar
outbox pattern no MySQL em vez de uma fila dedicada. Seções obrigatórias:
Status, Contexto, Decisão, Alternativas Consideradas, Consequências
(positivas E negativas, com trade-off explícito). Toda afirmação de
contexto ou decisão precisa citar o timestamp e o falante. Se alguma
afirmação não estiver nos trechos colados, não a inclua — diga que falta
informação em vez de completar com suposição.
```

## Iterações e ajustes
1. **Risco de inventar caminhos de arquivo no FDD.** A primeira tentativa de esboçar a seção "Integração com o sistema existente" citava nomes plausíveis, mas genéricos, de arquivos (ex. um hipotético `webhook.config.ts` já existente). Antes de escrever a versão final, usei `gh api` para confirmar a árvore real do repositório base e o conteúdo de cada arquivo citado — a seção final só referencia os 12 caminhos reais confirmados (`order.service.ts`, `order.status.ts`, `app-error.ts`, `http-errors.ts`, `errors/index.ts`, `auth.middleware.ts`, `error.middleware.ts`, `logger/index.ts`, `server.ts`, `routes/index.ts`, `schema.prisma`, `response.ts`).
2. **RFC duplicando detalhe de implementação do FDD.** Uma primeira passagem do RFC listava exemplos de payload JSON e a matriz completa de headers HTTP — detalhe que pertence ao FDD. Na revisão, esse conteúdo foi removido do RFC e mantido só como visão geral de uma linha por etapa do fluxo (outbox → worker → HMAC → retry/DLQ), com link explícito para o FDD em vez de repetição.
3. **Objetivo quantitativo do PRD sem origem clara.** A primeira versão do PRD trazia uma meta de "zero perda de eventos", que não tem origem identificável e nem é tecnicamente verdadeira (a garantia é at-least-once, não "zero perda" absoluta). Foi substituída por uma métrica de latência rastreável à reunião (p95 de entrega em até 10s, ancorado em [09:02] e na decisão de polling de 2s). A métrica de confiabilidade que a acompanhava passou por uma segunda correção na revalidação da iteração 5 abaixo — o primeiro substituto também continha um número sem origem real.
4. **Tracker com itens sem origem identificável.** Ao montar o Tracker, um código de erro defensivo interno (`WEBHOOK_EVENT_NOT_ELIGIBLE`, uma guarda que nunca deveria disparar em condições normais) não tinha nenhum timestamp ou arquivo de origem genuíno — em vez de forçar uma Localização inventada, ele foi deixado fora da tabela e documentado como exceção no topo do `TRACKER.md`.
5. **Revalidação linha a linha contra a transcrição (não só contra o enunciado).** Depois da primeira versão completa do pacote, foi feita uma segunda passada comparando cada citação `[hh:mm] Nome` com a fala exata naquele timestamp — e não apenas verificando se as seções obrigatórias existiam. Essa passada encontrou e corrigiu 6 problemas reais: (a) os FRs de PATCH/DELETE/GET de webhook estavam atribuídos a "Marcos" quando na verdade foi Bruno quem os enumerou em [09:33]; (b) a meta de "≥99% de entrega" no PRD era um número inventado sem origem — o único "99%" da reunião é sobre outra coisa (cobertura de casos do at-least-once, não uma SLA de entrega), então a métrica foi substituída por um objetivo verificável derivado da própria decisão de DLQ; (c) o número de tentativas de retry era inconsistente entre documentos — o ADR dizia "5 tentativas totais" enquanto o FDD já modelava corretamente 6 tentativas (1 inicial + 5 retries), confirmado pela própria conta de "quase 15 horas" que Diego faz em [09:17]; essa ambiguidade foi resolvida e documentada explicitamente no ADR-003; (d) uma citação no ADR-004 misturava, dentro de uma única aspa, falas de Sofia e de Bruno como se fossem de uma só pessoa; (e) a frase sobre autorização dos endpoints no FDD se contradizia ("ficam sob requireRole... sem restrição de role"); (f) o PRD promete rotação de secret (FR13/ADR-004) mas o FDD não tinha nenhum contrato HTTP para isso — foi adicionado o endpoint `POST /webhooks/:id/secret/rotate` que faltava, e isso também resolveu uma matriz de erro (`WEBHOOK_PAYLOAD_TOO_LARGE`) que contradizia o princípio de isolamento de falha do próprio documento.
6. **Documentos que descrevem outros documentos ficam desatualizados quando a fonte muda.** Corrigir o item 5 criou dois efeitos colaterais não percebidos na hora: o README ainda citava a métrica "≥99%" (já substituída) como se fosse a atual, e o Tracker/README ainda diziam "6 endpoints" / "10 arquivos reais" depois que o endpoint de rotação de secret elevou essas contagens para 7 e 11. Corrigido recontando tudo programaticamente em vez de de cabeça.
7. **Critério de aceite verificado mecanicamente, não só por leitura.** Rodando um script para contar quantos dos endpoints do FDD mostravam request *e* response, descobri que só 2 de 7 tinham o lado "request" documentado — abaixo do mínimo exigido de 4. Adicionei o request (corpo, query string ou "sem corpo") em todos. A mesma rodada também achou que `docs/adrs/README.md` (arquivo do boilerplate, não escrito por mim) ainda instruía nomear ADRs como `0001-titulo.md`, contradizendo os 7 arquivos `ADR-NNN-titulo.md` reais na mesma pasta — corrigido.
8. **A seção "Contratos públicos" nunca descrevia o contrato mais importante da feature.** O FDD dizia, na seção de Fluxos, que o worker envia "os headers descritos na seção de Contratos Públicos" — mas `X-Timestamp` e `X-Webhook-Id` nunca apareciam ali; só os endpoints de CRUD/admin estavam documentados, não a própria chamada de entrega do webhook (o `POST` outbound do worker para o cliente, com `X-Event-Id`/`X-Signature`/`X-Timestamp`/`X-Webhook-Id`). Adicionada uma subseção dedicada a esse contrato, com exemplo de payload e headers.
9. **Arquivo citado em "Contratos públicos" mas ausente da seção "Integração com o sistema existente".** O endpoint de listagem reaproveita `PaginatedResponse` de `src/shared/http/response.ts` — e o Tracker já tinha uma linha (`FDD-INTEGRACAO-10`) para esse arquivo — mas a própria seção de Integração do FDD parava no item 9 e nunca o citava, uma referência cruzada sem destino real. Adicionado como item 10 da seção. Revisão de citações linha a linha também achou dois timestamps errados por 1 minuto no Tracker (`FDD-FLUXO-01` e `FDD-FLUXO-03`, cada um citando a frase de um minuto antes de quando ela foi realmente dita) e uma atribuição de falante que dava todo o crédito a quem só fez a pergunta de escopo, não a quem decidiu a resposta (`PRD-SCOPE-OUT-01`).

## Como navegar a entrega
Ordem de leitura sugerida:
1. [`docs/PRD.md`](docs/PRD.md) — por quê e o quê (visão de produto, requisitos, escopo).
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica de arquitetura, para revisão, com alternativas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 7 decisões individuais (ADR-001 a ADR-007), cada uma referenciada a partir do RFC.
4. [`docs/FDD.md`](docs/FDD.md) — detalhamento de implementação: fluxos, contratos HTTP, matriz de erros, observabilidade e a seção de integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — tabela de rastreabilidade cruzando todo item acima com a `TRANSCRICAO.md` ou o código-fonte.

O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado nesta entrega — serve apenas de contexto e referência, conforme pedido no desafio original.
