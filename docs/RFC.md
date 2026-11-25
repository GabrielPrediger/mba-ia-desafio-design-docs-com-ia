# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
|---|---|
| **Autor** | Bruno (Engenheiro Pleno, time de Pedidos) — proposta consolidada a partir da reunião técnica de 09:00 |
| **Status** | Proposto para revisão |
| **Data** | Quinta-feira, reunião de alinhamento técnico (ver `TRANSCRICAO.md`) |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança) |

## Resumo Executivo (TL;DR)

Vamos construir um sistema de notificação assíncrona por webhook para avisar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) quando o status de um pedido muda, substituindo o polling atual deles sobre `GET /orders`. A mudança de status publica um evento transacional em uma tabela outbox no MySQL existente; um worker separado, em polling de 2 segundos, lê a outbox e entrega via HTTP com assinatura HMAC-SHA256, retry com backoff exponencial e DLQ para falhas permanentes. O módulo segue os padrões já estabelecidos no projeto (estrutura de módulo, `AppError`, Pino, autenticação JWT). Entrega estimada em três sprints, incluindo revisão de segurança da Sofia.

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram formalmente ser notificados em tempo real quando o status de seus pedidos muda na plataforma. Atualmente, eles fazem polling periódico em `GET /orders`, o que é lento, caro para a integração deles, e já gerou um risco comercial explícito: a Atlas sinalizou que pode migrar para um concorrente se a feature não for entregue até o fim do trimestre.

O requisito funcional de "tempo real" foi quantificado pelo PM como "qualquer coisa abaixo de 10 segundos". O sistema é puramente **outbound**: a plataforma envia notificações para os clientes; eles não enviam nada de volta por este canal.

A aplicação hoje não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks — este vácuo é o que a feature preenche.

## Proposta Técnica

A proposta tem três componentes:

**1. Captura do evento — padrão Outbox transacional.** Dentro da mesma transação que já executa a mudança de status em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) — que atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` conforme a transição — inserimos também uma linha em uma nova tabela `webhook_outbox`, com o payload do evento já renderizado (snapshot). Se a transação falhar, o evento nunca existiu; se commitar, o evento está garantido. Detalhe completo em [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md) e [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md).

**2. Entrega assíncrona — worker dedicado.** Um processo Node separado (`src/worker.ts`, script `npm run worker`), com sua própria instância de `PrismaClient`, faz polling a cada 2 segundos na `webhook_outbox`, processa os eventos pendentes mais antigos e os envia via HTTP para a URL cadastrada de cada webhook. Falhas acionam retry com backoff exponencial (5 tentativas: 1m/5m/30m/2h/12h); esgotadas as tentativas, o evento vai para `webhook_dead_letter`, reprocessável manualmente por um endpoint administrativo. Detalhe em [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md) e [ADR-005](adrs/ADR-005-worker-em-processo-separado-com-polling.md).

**3. Segurança e contrato de entrega.** Cada requisição carrega os headers `X-Event-Id` (UUID, para deduplicação pelo cliente — garantimos *at-least-once*, não *exactly-once*), `X-Signature` (HMAC-SHA256 do corpo, com secret única por endpoint e rotação com grace period de 24h), `X-Timestamp` e `X-Webhook-Id`. URLs de webhook devem ser HTTPS; payload limitado a 64KB; timeout de 10s por tentativa. Detalhe em [ADR-003](adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md) e [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md).

O módulo novo (`src/modules/webhooks/`) segue integralmente os padrões já estabelecidos no projeto: estrutura `controller`/`service`/`repository`/`routes`/`schemas`, erros derivados de `AppError` com prefixo `WEBHOOK_*`, logging via Pino, autenticação JWT existente. Ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md). O detalhamento de endpoints, fluxos e contratos fica no [FDD](FDD.md), não neste documento.

## Alternativas Consideradas

1. **Disparo síncrono dentro da transação de `changeStatus`.** Foi a primeira opção discutida. Descartada porque acoplaria a confiabilidade de uma operação interna crítica (mudança de status, já uma transação pesada envolvendo estoque) à disponibilidade e latência de um sistema de terceiros — e não há como dar rollback em um pedido que já mudou de estado de fato se o cliente externo estiver fora do ar. Trade-off: simplicidade de implementação vs. risco de travar o core do sistema por causa de um cliente externo lento ou indisponível.

2. **Fila dedicada (Redis Streams ou similar) em vez de outbox no MySQL.** Resolveria o mesmo problema de desacoplamento, mas exigiria subir e operar uma peça de infraestrutura nova para um time pequeno. Descartada como overengineering frente ao volume atual e ao requisito de latência (abaixo de 10s), que um polling simples de 2s já atende com folga. Trade-off: a fila dedicada escalaria melhor no longo prazo, mas o custo operacional imediato não se justifica agora.

## Questões em Aberto

1. **Rate limiting de saída para o cliente.** Se um cliente tiver, por exemplo, 50 pedidos mudando de status no mesmo minuto, hoje o worker dispara 50 chamadas para ele sem nenhum controle de taxa. Ficou decidido observar o comportamento em produção e decidir sobre uma implementação apenas se isso se mostrar um problema real.
2. **Notificação proativa de webhook com falhas recorrentes.** A ideia de avisar o cliente (ex.: por e-mail) quando o webhook dele falha repetidamente (ex.: 3 falhas seguidas) foi levantada e explicitamente adiada para uma fase futura, condicionada a medirmos o impacto real da ausência dessa notificação nesta primeira versão.

## Impacto e Riscos

- **Risco comercial mitigado**: entrega end-to-end em até três sprints atende o prazo sinalizado pela Atlas (fim de novembro), reduzindo o risco de perda do cliente.
- **Risco de segurança**: exposição de dados de pedidos para sistemas de terceiros exige a camada de assinatura HMAC e TLS obrigatório (ver ADR-003); Sofia reservou pelo menos dois dias úteis de revisão de segurança dedicada ao código de HMAC e geração de secret antes do deploy.
- **Risco operacional**: um novo processo (`worker`) para subir, monitorar e fazer deploy; mitigado por reaproveitar integralmente a stack e os padrões já operados pelo time (mesmo Prisma, mesmo MySQL, mesmo logger).
- **Risco de dados**: payload de webhook carrega dados de pedido (incluindo `total_cents`) para fora da infraestrutura; mitigado por não incluir itens detalhados do pedido no payload (cliente busca detalhe via `GET /orders/:id` se precisar) e por limitar o payload a 64KB.

## Decisões Relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-pattern-no-mysql.md)
- [ADR-002 — Retry com backoff exponencial e DLQ](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-003 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-004 — Garantia at-least-once com X-Event-Id](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)
- [ADR-005 — Worker em processo separado com polling](adrs/ADR-005-worker-em-processo-separado-com-polling.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
- [ADR-007 — Snapshot do payload na inserção do evento](adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md)
- [ADR-008 — Filtragem de eventos na inserção da outbox](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)
