# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status
Aceito

## Contexto
Os eventos de webhook saem da nossa infraestrutura para sistemas de terceiros (clientes B2B). Sofia (Segurança) levantou a necessidade de o cliente conseguir "validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio" [09:19 Sofia]. Era preciso escolher um mecanismo de assinatura e decidir o escopo de cada secret (global vs. por integração), considerando ainda um incidente real do passado: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez" [09:22 Diego].

## Decisão
Cada requisição de webhook é assinada com **HMAC-SHA256** sobre o corpo (payload) da requisição, e a assinatura é enviada no header `X-Signature`: "Padrão é HMAC. A gente assina o payload com uma secret compartilhada entre nós e o cliente, manda a assinatura num header tipo X-Signature. Cliente verifica do lado dele" [09:20 Sofia]. Perguntado por Bruno sobre qual algoritmo usar [09:20 Bruno], Sofia respondeu: "SHA-256. HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso" [09:20 Sofia].

Cada **endpoint de webhook cadastrado tem uma secret própria**, não uma secret global da plataforma: "cada endpoint de webhook do cliente tem que ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo" [09:21 Sofia]. A secret é gerada pelo sistema e devolvida uma única vez na criação do webhook [09:31 Marcos].

A secret é **rotacionável** via API, e durante a rotação a secret antiga permanece válida em paralelo por um **grace period de 24 horas**, para o cliente migrar sem downtime: "quando ele rotaciona, a antiga fica válida por 24 horas em paralelo [...] Depois disso, a antiga morre" [09:21-22 Sofia].

## Alternativas Consideradas
**Secret global única por plataforma.** Implicitamente levantada e descartada em favor de secret por endpoint, justamente pelo risco de propagação de um vazamento: uma única secret vazada comprometeria a autenticidade de **todos** os webhooks da plataforma, não só de um cliente [09:21 Sofia]. O trade-off foi complexidade de gestão (uma secret por endpoint, com necessidade de rotação individual) vs. isolamento de blast radius em caso de vazamento.

## Consequências
**Positivas:**
- Vazamento de uma secret compromete apenas um endpoint/cliente, não a plataforma inteira.
- Rotação com grace period evita downtime de integração durante troca de secret.
- HMAC-SHA256 é padrão amplamente suportado por bibliotecas client-side, reduzindo fricção de integração para os clientes B2B.

**Negativas:**
- Mais secrets para gerenciar e armazenar de forma segura (uma por endpoint, mais a secret antiga durante o grace period de 24h).
- Exige lógica adicional de expiração/invalidação da secret antiga após o grace period (job ou verificação no momento da assinatura).
- A revisão de segurança da Sofia sobre geração e armazenamento de secret é um gate explícito antes do deploy, reservando no mínimo 2 dias úteis [09:46 Sofia] — restrição de cronograma.
