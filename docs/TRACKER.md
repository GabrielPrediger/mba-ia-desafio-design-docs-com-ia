# Tracker de Rastreabilidade

Esta tabela mapeia cada item relevante registrado no PRD, no RFC, no FDD e nos ADRs à sua origem: um trecho específico de `TRANSCRICAO.md` (timestamp + falante) ou um caminho de arquivo real no código base. O objetivo é permitir que qualquer leitor confira a procedência de cada requisito, decisão ou restrição, e servir como defesa contra alucinação da IA — qualquer linha sem origem identificável aqui é um sinal de que o conteúdo correspondente deve ser revisado ou removido.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com URL (HTTPS) e lista de status de interesse | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida só na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | PATCH para editar webhook (url, events, active) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | GET para listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Publicação de evento somente para webhooks com filtro correspondente | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retry com 5 tentativas e backoff exponencial | TRANSCRICAO | [09:15] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Falha permanente move evento para Dead Letter Queue | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Replay manual de DLQ restrito a role ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Payload enxuto, sem itens detalhados do pedido | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Limite de 64KB por payload, erro acima disso | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | URL de webhook obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Reuso dos padrões arquiteturais existentes, sem convenções novas | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova (sem Redis/fila dedicada) | TRANSCRICAO | [09:07] Diego |
| PRD-SCOPE-OUT-01 | docs/PRD.md | Restrição | Notificação proativa (e-mail) sobre webhook com falha recorrente adiada | TRANSCRICAO | [09:37] Larissa |
| PRD-SCOPE-OUT-02 | docs/PRD.md | Restrição | Dashboard visual para o cliente fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-SCOPE-OUT-03 | docs/PRD.md | Restrição | Rate limiting de saída não implementado nesta fase | TRANSCRICAO | [09:39] Larissa |
| PRD-SCOPE-OUT-04 | docs/PRD.md | Restrição | Sistema estritamente outbound, sem webhooks de entrada | TRANSCRICAO | [09:02] Marcos |
| PRD-SCOPE-OUT-05 | docs/PRD.md | Restrição | Arquivamento automático de eventos antigos fora de escopo | TRANSCRICAO | [09:08] Diego |
| PRD-OBJ-01 | docs/PRD.md | Decisão | Meta de latência: abaixo de 10s é "tempo real" para os clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Decisão | Prazo de entrega estimado em três sprints, incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Decisão | Garantia de zero perda silenciosa via outbox transacional | TRANSCRICAO | [09:06] Diego |
| PRD-RISK-01 | docs/PRD.md | Trade-off | Indisponibilidade do cliente pode superar a janela de retry (~15h) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-02 | docs/PRD.md | Trade-off | Vazamento de secret já ocorreu uma vez no passado | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Trade-off | Cliente pode implementar deduplicação de forma incorreta | TRANSCRICAO | [09:25] Sofia |
| RFC-ALT-01 | docs/RFC.md | Decisão | Disparo síncrono no changeStatus descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Decisão | Fila dedicada (Redis Streams) descartada por overengineering | TRANSCRICAO | [09:07] Larissa |
| RFC-OPEN-01 | docs/RFC.md | Restrição | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Restrição | Notificação de falha recorrente adiada para fase futura | TRANSCRICAO | [09:38] Marcos |
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão | Padrão Outbox transacional no MySQL existente | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md | Decisão | Retry com 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela própria | TRANSCRICAO | [09:15] Diego |
| ADR-003 | docs/adrs/ADR-003-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret única por endpoint e rotação com grace period | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-com-x-event-id.md | Decisão | Entrega at-least-once com dedup via X-Event-Id | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo Node separado, polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso de AppError, Pino, auth middleware e padrão de módulos | CODIGO | src/shared/errors/app-error.ts |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Decisão | Payload renderizado como snapshot no momento da inserção do evento | TRANSCRICAO | [09:52] Larissa |
| ADR-008 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Decisão | Filtro de eventos de interesse aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Criação do evento dentro da transação de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-FLUXO-02 | docs/FDD.md | Requisito Funcional | Worker processa lote de pendentes a cada ciclo de polling | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Requisito Funcional | Progressão de backoff aplicada a partir da tentativa atual | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Requisito Funcional | Replay de DLQ recoloca evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | POST /webhooks — criação, customerId no body, não no JWT | TRANSCRICAO | [09:32] Larissa |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | GET /webhooks (lista por customerId) | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | POST /webhooks/:id/rotate-secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | POST /admin/webhooks/dead-letter/:id/replay (role ADMIN) | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-01 | docs/FDD.md | Decisão | Padrão de classe de erro para WEBHOOK_NOT_FOUND segue NotFoundError existente | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-02 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_INVALID_URL para URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERR-03 | docs/FDD.md | Decisão | Prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-04 | docs/FDD.md | Requisito Não Funcional | WEBHOOK_PAYLOAD_TOO_LARGE acima de 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERR-05 | docs/FDD.md | Requisito Funcional | WEBHOOK_DEAD_LETTER_NOT_FOUND no replay inválido | TRANSCRICAO | [09:18] Diego |
| FDD-ERR-06 | docs/FDD.md | Decisão | Erro de entrega tratado pelo error.middleware já existente | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRACAO-01 | docs/FDD.md | Decisão | Extensão de changeStatus para chamar publishWebhookEvent(tx, ...) | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEGRACAO-02 | docs/FDD.md | Decisão | Novas classes de erro estendem AppError/http-errors existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEGRACAO-03 | docs/FDD.md | Decisão | errorMiddleware absorve erros WEBHOOK_* sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEGRACAO-04 | docs/FDD.md | Decisão | Reuso de authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEGRACAO-05 | docs/FDD.md | Decisão | Worker replica padrão de criação de PrismaClient | CODIGO | src/config/database.ts |
| FDD-INTEGRACAO-06 | docs/FDD.md | Decisão | Novas tabelas seguem convenções de UUID/@@map do schema existente | CODIGO | prisma/schema.prisma |

**Cobertura:** 64 itens rastreados; 54 com Fonte = TRANSCRICAO (84%, todos com timestamp `[hh:mm] Nome` válido) e 10 com Fonte = CODIGO (caminho de arquivo real), superando os mínimos exigidos (≥80% de cobertura geral, ≥70% com timestamp de transcrição válido, ≥5 linhas com Fonte = CODIGO).
