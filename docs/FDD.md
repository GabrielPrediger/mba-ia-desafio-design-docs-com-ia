# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

O OMS precisa notificar sistemas de clientes B2B sobre mudanças de status de pedido, substituindo o polling atual em `GET /orders`. Este documento especifica, em nível acionável, como o módulo `src/modules/webhooks` e o worker `src/worker.ts` devem ser construídos, consumindo as decisões já fechadas nos [ADRs](adrs/) e na proposta do [RFC](RFC.md). O FDD não repete o "porquê" das decisões (isso está no RFC/ADRs); ele especifica o "como".

## Objetivos Técnicos

- Publicar um evento de forma transacional e atômica junto a toda mudança de status de pedido, sem exceção (ADR-001).
- Entregar eventos a endpoints HTTPS externos com latência observada abaixo de 10s na maioria dos casos (polling de 2s, ADR-005).
- Garantir que nenhum evento seja perdido silenciosamente: toda falha permanente termina em DLQ auditável (ADR-002).
- Garantir autenticidade e integridade do payload do lado do cliente via HMAC-SHA256 (ADR-003).
- Não introduzir nenhuma tecnologia de infraestrutura nova além do MySQL/Prisma já existente.

## Escopo e Exclusões

**Dentro do escopo:** CRUD de configuração de webhook, publicação transacional de eventos, worker de entrega com retry/DLQ, endpoint de histórico de entregas, endpoint administrativo de replay de DLQ.

**Fora do escopo (ver PRD para detalhamento completo):** notificação proativa ao cliente sobre webhook com falha recorrente (ex.: e-mail), rate limiting de saída, dashboard visual para o cliente, arquivamento automático de eventos antigos da outbox, suporte a webhooks de entrada (inbound).

## Fluxos Detalhados

### Fluxo 1 — Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa a transação já existente: valida transição (`canTransition`), debita/repõe estoque se aplicável, atualiza `orders.status`, insere em `order_status_history`.
2. Antes do commit da mesma transação (`tx`), o service chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` — uma função pura do módulo de webhooks que recebe o `tx` client ativo (não um repository inteiro injetado).
3. `publishWebhookEvent` busca, dentro do mesmo `tx`, os webhooks ativos do `customer_id` do pedido cujo filtro de eventos inclui `toStatus` (ver ADR-008).
4. Para cada webhook encontrado, monta o payload (snapshot, ver Fluxo 1 — Contratos) e insere uma linha em `webhook_outbox` com `status = 'pending'`, `event_id` (UUID novo), `webhook_id`, `payload` (JSON já renderizado) e `created_at`.
5. Se nenhum webhook do cliente quiser aquele status, nenhuma linha é inserida (sem efeito colateral).
6. Se qualquer etapa falhar, toda a transação (incluindo a mudança de status original) sofre rollback — não existe caso de status mudar sem o evento correspondente ser registrado.

### Fluxo 2 — Processamento pelo worker

1. `src/worker.ts` inicializa seu próprio `PrismaClient` (padrão de `src/config/database.ts`) e entra em loop de polling a cada 2 segundos.
2. A cada ciclo, busca um lote (ex.: 20) de linhas em `webhook_outbox` com `status = 'pending'` e `next_attempt_at <= now()`, ordenadas por `created_at` ascendente.
3. Para cada linha, marca como `status = 'processing'`, monta os headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e faz o POST para a `url` do webhook com timeout de 10s.
4. Em caso de resposta HTTP 2xx: marca a linha como `status = 'delivered'`, `delivered_at = now()`, registra em `webhook_delivery_log` (sucesso, status code, tempo de resposta).
5. Em caso de erro (timeout, conexão recusada, HTTP 4xx/5xx): incrementa `attempt_count`, registra em `webhook_delivery_log` (falha, motivo), calcula `next_attempt_at` pela tabela de backoff (ADR-002) e volta `status = 'pending'`.
6. Se `attempt_count` atingir 5 e a última tentativa falhar: move a linha para `webhook_dead_letter` (payload, motivo, timestamp) e remove (ou marca como `failed` terminal) a linha da outbox.

### Fluxo 3 — Retry e backoff

Tabela de intervalos (a partir do `attempt_count` atual): 1ª retentativa após 1 minuto, 2ª após 5 minutos, 3ª após 30 minutos, 4ª após 2 horas, 5ª após 12 horas. Após a 5ª tentativa falhar, vai para DLQ. `next_attempt_at` é sempre calculado a partir do momento da falha atual (`now() + intervalo`), não do `created_at` original.

### Fluxo 4 — DLQ e replay manual

1. Um registro em `webhook_dead_letter` contém: `id` (UUID), `webhook_id`, `event_id` original, `payload`, `failure_reason`, `attempt_count`, `created_at`.
2. Um operador ADMIN chama `POST /admin/webhooks/dead-letter/:id/replay`.
3. O handler valida `requireRole('ADMIN')`, busca o registro da DLQ, recria uma linha em `webhook_outbox` com `status = 'pending'`, `attempt_count = 0`, novo `event_id` (ou preserva o original — decisão de implementação não crítica para o contrato externo) e remove (ou marca como `replayed`) o registro da DLQ.
4. A ação é logada via Pino com `userId` do operador, `dead_letter_id` e timestamp, para fins de auditoria (requisito de segurança da Sofia).

## Contratos Públicos

Todos os endpoints abaixo vivem sob `src/modules/webhooks/webhook.routes.ts`, montados em `/api/v1/webhooks` (seguindo o padrão de `src/routes/index.ts`), exceto o endpoint de replay, que vive sob `/api/v1/admin/webhooks`.

### 1. `POST /api/v1/webhooks`

Cria um webhook. Autenticado (qualquer role).

Request:
```json
{
  "customerId": "c3b1a2e0-1111-4a2b-9c1d-000000000001",
  "url": "https://atlas-integracao.example.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "f1e2d3c4-2222-4a2b-9c1d-000000000002",
  "customerId": "c3b1a2e0-1111-4a2b-9c1d-000000000001",
  "url": "https://atlas-integracao.example.com/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_7f3a1c9e8b2d4f6a0e1c9b7d5a3f2e1c",
  "active": true,
  "createdAt": "2026-11-18T13:04:00.000Z"
}
```
A `secret` só é retornada nesta resposta de criação (e na resposta de rotação); nunca é devolvida em `GET`.

### 2. `PATCH /api/v1/webhooks/:id`

Atualiza `url`, `events` ou `active`. Autenticado.

Request:
```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```
Response `200 OK`: mesmo shape do `GET`, sem `secret`.

### 3. `DELETE /api/v1/webhooks/:id`

Remove um webhook. Autenticado. Response `204 No Content`.

### 4. `GET /api/v1/webhooks?customerId=...`

Lista webhooks de um customer. Autenticado.

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "f1e2d3c4-2222-4a2b-9c1d-000000000002",
      "customerId": "c3b1a2e0-1111-4a2b-9c1d-000000000001",
      "url": "https://atlas-integracao.example.com/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-18T13:04:00.000Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

### 5. `POST /api/v1/webhooks/:id/rotate-secret`

Gera uma nova secret para o endpoint, mantendo a anterior válida por 24h (ADR-003). Autenticado.

Response `200 OK`:
```json
{
  "id": "f1e2d3c4-2222-4a2b-9c1d-000000000002",
  "secret": "whsec_a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "previousSecretValidUntil": "2026-11-19T13:04:00.000Z"
}
```

### 6. `GET /api/v1/webhooks/:id/deliveries`

Histórico das últimas 100 entregas (sucesso/falha, payload enviado, resposta, tempo de resposta). Autenticado.

Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "9c1d0002-3333-4a2b-9c1d-000000000003",
      "status": "delivered",
      "httpStatus": 200,
      "responseTimeMs": 184,
      "attempt": 1,
      "sentAt": "2026-11-18T13:05:02.000Z"
    },
    {
      "eventId": "9c1d0003-4444-4a2b-9c1d-000000000004",
      "status": "failed",
      "httpStatus": 503,
      "responseTimeMs": 10000,
      "attempt": 3,
      "sentAt": "2026-11-18T14:10:00.000Z"
    }
  ]
}
```

### 7. `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Reprocessa manualmente um item da DLQ. Requer `requireRole('ADMIN')`.

Response `200 OK`:
```json
{
  "deadLetterId": "0a1b2c3d-5555-4a2b-9c1d-000000000005",
  "requeuedEventId": "0a1b2c3d-6666-4a2b-9c1d-000000000006",
  "status": "pending"
}
```

## Matriz de Erros (prefixo `WEBHOOK_`)

| Código | HTTP Status | Situação |
|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook com o `id` informado não existe. |
| `WEBHOOK_INVALID_URL` | 400 | URL cadastrada não é HTTPS ou não é uma URL válida (validação Zod). |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Operação exige uma secret ativa e nenhuma foi encontrada (ex.: estado inconsistente). |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload renderizado excede 64KB; evento não é inserido na outbox. |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Registro de DLQ informado no replay não existe. |
| `WEBHOOK_DELIVERY_FAILED` | — (uso interno/log) | Código de log interno do worker para uma tentativa de entrega malsucedida; não é retornado via HTTP a um cliente da nossa API. |

Todos seguem o mesmo contrato de resposta de erro já padronizado pelo `errorMiddleware` (`src/middlewares/error.middleware.ts`): `{ "error": { "code", "message", "details"? } }`.

## Estratégias de Resiliência

- **Timeout**: 10 segundos por tentativa de entrega HTTP (acima disso, trata como falha e segue para retry).
- **Retry**: backoff exponencial com 5 tentativas (1m/5m/30m/2h/12h) — ver Fluxo 3.
- **DLQ**: isolamento de falhas permanentes em tabela própria, com replay manual controlado por role ADMIN.
- **Fallback**: nenhum fallback automático de canal (ex.: e-mail) nesta fase — fora de escopo (ver PRD).
- **Transação atômica**: a inserção do evento nunca ocorre fora da transação de domínio que originou a mudança de status (ADR-001).

## Observabilidade

- **Métricas** (expostas via log estruturado Pino, agregáveis pela stack de observabilidade existente): contagem de eventos inseridos na outbox por status de destino; contagem de entregas por resultado (sucesso/falha) e por tentativa; tamanho atual da fila de pendentes (`webhook_outbox` com `status = 'pending'`); contagem de itens movidos para DLQ; latência de entrega (tempo entre `created_at` do evento e `delivered_at`).
- **Logs**: o worker loga cada tentativa de entrega (sucesso ou falha) via o logger Pino já existente (`src/shared/logger/index.ts`), incluindo `event_id`, `webhook_id`, `attempt`, resultado e tempo de resposta. O replay administrativo loga o `userId` do operador para auditoria.
- **Tracing**: cada evento carrega seu `event_id` (UUID) desde a inserção até a entrega final (ou DLQ), funcionando como correlation ID natural entre os logs do `OrderService`, do worker e do histórico de entregas — sem necessidade de uma ferramenta de tracing distribuído dedicada nesta fase.

## Integração com o Sistema Existente

Esta seção nomeia os pontos reais de integração no código base:

1. **`src/modules/orders/order.service.ts` (método `changeStatus`)**: é estendido para, dentro da mesma transação (`tx`), chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` logo após a atualização de `orders.status` e a inserção em `order_status_history`, e antes do `return` do bloco de transação. Nenhuma outra lógica do método é alterada.
2. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`**: as novas classes de erro do módulo de webhooks (ex.: `WebhookNotFoundError`, `WebhookInvalidUrlError`) estendem `AppError` (ou as classes HTTP já existentes como `NotFoundError`/`UnprocessableEntityError`), exatamente como `InsufficientStockError` e `InvalidStatusTransitionError` já fazem hoje para o domínio de pedidos.
3. **`src/middlewares/error.middleware.ts`**: não sofre nenhuma alteração — por já tratar genericamente qualquer instância de `AppError`, `ZodError` e erros conhecidos do Prisma, absorve automaticamente os novos erros `WEBHOOK_*` assim que eles são lançados pelo novo módulo.
4. **`src/middlewares/auth.middleware.ts`**: o módulo de webhooks reutiliza diretamente `authenticate` (em todas as rotas) e `requireRole('ADMIN')` (apenas na rota de replay de DLQ), sem nenhuma extensão do middleware.
5. **`src/config/database.ts`**: o worker (`src/worker.ts`) replica o padrão de criação de `PrismaClient` definido aqui (mesma configuração de log por ambiente), mas instanciando seu próprio client — por rodar em processo Node separado do da API.
6. **`prisma/schema.prisma`**: são adicionados os novos models (`WebhookEndpoint`, `WebhookOutbox`, `WebhookDeadLetter`, `WebhookDeliveryLog`), seguindo as convenções já usadas (`@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, índices em colunas de filtro frequente como `status` e `createdAt`, nos mesmos moldes de `Order`/`OrderStatusHistory`).

## Dependências e Compatibilidade

- Nenhuma dependência de infraestrutura nova (sem Redis, sem broker de mensageria). Dependências de código novas, se necessárias (ex.: biblioteca de HMAC), usam o que o Node já oferece nativamente (`crypto` do Node já implementa HMAC-SHA256) — nenhuma dependência externa nova é estritamente necessária para a assinatura.
- Compatível com o schema de banco existente: as novas tabelas não alteram nenhuma tabela atual (`orders`, `products`, etc.), apenas referenciam `customer_id` como chave estrangeira lógica.
- `package.json`: adiciona o script `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e seu equivalente de build/start, seguindo o padrão já usado por `"dev"`/`"start"`.

## Critérios de Aceite Técnicos

- Nenhuma mudança de status de pedido ocorre sem a correspondente avaliação de publicação de evento (mesma transação).
- Eventos pendentes são processados pelo worker em no máximo ~2 segundos de atraso médio, respeitando o requisito de "abaixo de 10s" percebido pelo cliente.
- Após 5 tentativas falhas, o evento aparece em `webhook_dead_letter` e desaparece da fila ativa de pendentes.
- Toda requisição de entrega carrega `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- URLs não-HTTPS são rejeitadas na validação de schema, antes de qualquer persistência.
- Payloads acima de 64KB nunca chegam a ser inseridos na outbox (erro `WEBHOOK_PAYLOAD_TOO_LARGE`).

## Riscos e Mitigação

- **Risco**: worker parado (crash, deploy) acumula eventos pendentes sem processar. **Mitigação**: processo independente supervisionado como qualquer outro processo de produção do time (fora do escopo desta feature detalhar a ferramenta de supervisão, mas a arquitetura não impede reinício seguro — eventos pendentes continuam na tabela).
- **Risco**: vazamento de secret do lado do cliente. **Mitigação**: rotação com grace period de 24h (ADR-003), permitindo resposta rápida sem downtime para o cliente.
- **Risco**: cliente mal implementa deduplicação por `X-Event-Id` e trata duplicatas como pedidos novos. **Mitigação**: documentação destacada no portal do desenvolvedor (responsabilidade do PM, fora do código).
