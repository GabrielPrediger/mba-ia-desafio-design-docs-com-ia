# PRD — Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O Order Management System (OMS) hoje não possui nenhum mecanismo de notificação externa: clientes que querem saber quando o status de um pedido muda precisam consultar `GET /orders` repetidamente. Esta feature introduz um sistema de **webhooks outbound**: sempre que o status de um pedido muda, o sistema notifica, via HTTP, os endpoints previamente cadastrados pelos clientes interessados naquele status.

## Problema e Motivação

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram formalmente ser avisados em tempo real sobre mudanças de status de seus pedidos. Hoje eles fazem polling periódico em `GET /orders`, o que é lento e caro para a integração do lado deles. Esse atrito já gerou um risco comercial concreto: a Atlas sinalizou que pode migrar para um concorrente caso a feature não seja entregue até o fim do trimestre (prazo confirmado para fim de novembro).

## Público-Alvo e Cenários de Uso

- **Clientes B2B integrados via API** (inicialmente Atlas Comercial, MaxDistribuição, Nova Cargo): cadastram um ou mais endpoints de webhook, escolhem quais status de pedido querem acompanhar, e recebem notificações HTTP assinadas sempre que esses status ocorrem para seus pedidos.
- **Operadores internos (usuários autenticados do OMS)**: realizam o cadastro e a manutenção dos webhooks dos clientes via API autenticada (o cliente final não acessa diretamente o OMS — quem cadastra é um usuário interno em nome do cliente, ou futuramente um painel do cliente, fora de escopo nesta fase).
- **Administradores (role `ADMIN`)**: acompanham falhas de entrega persistentes via a Dead Letter Queue e podem reprocessar manualmente eventos que esgotaram as tentativas automáticas.

## Objetivos e Métricas de Sucesso

- Substituir o polling atual dos três clientes B2B por notificação assíncrona, com **latência de notificação abaixo de 10 segundos** em pelo menos 95% das mudanças de status entregues com sucesso na primeira tentativa — a meta quantitativa definida pelos próprios clientes como "tempo real".
- Entregar a feature completa (modelagem, worker, CRUD, segurança revisada) em até **três sprints**, incluindo a revisão de segurança dedicada da engenheira de segurança antes do deploy.
- Garantir **zero perda silenciosa de eventos**: toda mudança de status elegível resulta em um evento rastreável até a entrega bem-sucedida ou até um registro de falha permanente na Dead Letter Queue.

## Escopo

### Incluso

- Publicação transacional de evento de mudança de status (padrão outbox).
- Worker assíncrono dedicado para entrega via HTTP, com retry e backoff.
- Dead Letter Queue para falhas permanentes, com reprocessamento manual via endpoint administrativo.
- Assinatura HMAC-SHA256 dos payloads, com secret por endpoint e rotação com grace period.
- CRUD de configuração de webhook (criar, editar, remover, listar) por cliente.
- Endpoint de histórico de entregas (últimas 100) por webhook.
- Garantia de entrega at-least-once com deduplicação por `X-Event-Id` do lado do cliente.

### Fora de Escopo

- **Notificação proativa ao cliente sobre webhook com falhas recorrentes** (ex.: e-mail após 3 falhas seguidas) — adiado para uma fase futura, condicionado à medição do impacto real desta primeira versão.
- **Dashboard visual para o cliente acompanhar seus webhooks** — meramente endpoints de API nesta fase; um painel visual é projeto separado do time de frontend.
- **Rate limiting de envio** para clientes com alto volume de mudanças de status simultâneas — fica apenas como ponto de observação; será implementado somente se se mostrar um problema real em produção.
- **Webhooks de entrada (inbound)** — o sistema é estritamente outbound; clientes não enviam eventos para o OMS por este canal.
- **Arquivamento automático de eventos antigos da outbox** — rotina de limpeza/retenção fica fora do escopo desta feature.

## Requisitos Funcionais

1. Cliente deve poder cadastrar um webhook informando URL (HTTPS obrigatório) e a lista de status de pedido que deseja acompanhar.
2. O sistema deve gerar e devolver uma secret única no momento da criação do webhook, nunca reexibida depois.
3. Cliente deve poder editar (`PATCH`) a URL, a lista de eventos ou o estado ativo/inativo de um webhook já cadastrado.
4. Cliente deve poder remover (`DELETE`) um webhook cadastrado.
5. Cliente deve poder listar (`GET`) todos os webhooks cadastrados para um `customerId`.
6. Cliente deve poder rotacionar a secret de um webhook, mantendo a secret anterior válida por 24 horas em paralelo.
7. Cliente deve poder consultar o histórico das últimas 100 entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta).
8. O sistema deve publicar um evento de notificação sempre que o status de um pedido mudar para um status que algum webhook ativo do cliente dono do pedido tenha na sua lista de interesse.
9. O sistema deve tentar entregar cada evento até 5 vezes, com backoff exponencial (1m/5m/30m/2h/12h), antes de considerá-lo falha permanente.
10. Eventos que esgotarem as tentativas automáticas devem ser movidos para uma Dead Letter Queue persistida e auditável.
11. Um administrador (role `ADMIN`) deve poder reprocessar manualmente um evento da Dead Letter Queue via endpoint dedicado.
12. Toda requisição de entrega deve carregar uma assinatura HMAC-SHA256 do corpo no header `X-Signature`, permitindo ao cliente validar autenticidade e integridade.

## Requisitos Não Funcionais

- Payload de cada evento deve ser enxuto, sem incluir os itens detalhados do pedido (apenas identificadores e campos agregados como `total_cents`); detalhes completos continuam disponíveis via `GET /orders/:id`.
- Payload de um evento não pode exceder 64KB; eventos que excederem esse limite não são enviados.
- Timeout de 10 segundos por tentativa de entrega HTTP.
- URLs de webhook devem ser obrigatoriamente HTTPS; URLs HTTP são rejeitadas na validação.
- O módulo deve seguir os mesmos padrões arquiteturais já estabelecidos no projeto (estrutura de módulo, tratamento de erros, logging, autenticação), sem introduzir convenções novas.
- Nenhuma infraestrutura nova (fila dedicada, cache, etc.) deve ser introduzida; a solução reaproveita o MySQL/Prisma já existente.

## Decisões e Trade-offs Principais

- **Outbox transacional no MySQL em vez de fila dedicada**: evita subir infraestrutura nova para um time pequeno, ao custo de uma latência mínima de polling (ver [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md)).
- **At-least-once em vez de exactly-once**: mais simples de implementar, exige que o cliente deduplique pelo `X-Event-Id` (ver [ADR-004](adrs/ADR-004-garantia-at-least-once-com-x-event-id.md)).
- **5 tentativas com backoff de até 12h em vez de retry indefinido ou agressivo (3 tentativas)**: equilibra tolerância a indisponibilidades reais do cliente (já observadas na prática) com um ponto de corte claro para intervenção manual (ver [ADR-002](adrs/ADR-002-retry-com-backoff-exponencial-e-dlq.md)).

## Dependências

- Módulos existentes `orders` (ponto de integração para disparo do evento) e a infraestrutura compartilhada (`AppError`, Pino, autenticação JWT, Prisma/MySQL).
- Disponibilidade da engenheira de segurança (Sofia) para revisão dedicada do código de HMAC e geração de secret antes do deploy (pelo menos dois dias úteis reservados).
- Comunicação do time de produto com os clientes sobre o modelo de deduplicação (`X-Event-Id`) e sobre a política de rotação de secret, via portal de desenvolvedor.

## Riscos e Mitigação

1. **Risco**: indisponibilidade do cliente externo pode se estender além da janela de retry (~15h), resultando em evento não entregue automaticamente. **Probabilidade**: baixa a média (já houve um caso de ~2h, dentro da janela coberta). **Impacto**: médio (cliente não recebe notificação automática, mas o pedido em si não é afetado). **Mitigação**: Dead Letter Queue com reprocessamento manual disponível para qualquer evento que precise ser reenviado fora da janela automática.
2. **Risco**: vazamento de secret do lado do cliente (já ocorreu uma vez no passado com outro mecanismo). **Probabilidade**: baixa, mas não nula. **Impacto**: alto (permitiria forjar assinaturas válidas para aquele endpoint). **Mitigação**: secret única por endpoint (não global) e rotação com grace period de 24h, permitindo resposta rápida a um incidente sem causar indisponibilidade para o cliente.
3. **Risco**: cliente não implementa corretamente a deduplicação por `X-Event-Id` e trata entregas duplicadas como novos eventos de negócio. **Probabilidade**: média. **Impacto**: médio (depende da lógica de negócio do cliente). **Mitigação**: documentação destacada no portal de desenvolvedor sobre a garantia at-least-once e a necessidade de deduplicação.

## Critérios de Aceitação

- Um cliente com um webhook ativo para o status `SHIPPED` recebe uma notificação HTTP assinada em até poucos segundos após um pedido dele mudar para `SHIPPED`.
- Um cliente sem webhook cadastrado para determinado status não gera nenhum evento de outbox para esse status (sem overhead desnecessário).
- Após 5 tentativas falhas de entrega, o evento aparece na Dead Letter Queue e some da fila de processamento ativo do worker.
- Um administrador consegue reprocessar manualmente um evento da Dead Letter Queue e vê-lo ser entregue (ou falhar novamente) de forma rastreável.
- Toda notificação enviada contém `X-Event-Id`, `X-Signature`, `X-Timestamp` e `X-Webhook-Id`, e o cliente consegue validar a assinatura com a secret vigente.
- Nenhuma URL HTTP (não segura) é aceita no cadastro ou edição de um webhook.

## Estratégia de Testes e Validação

- **Testes unitários**: validação das regras de filtro de eventos (ADR-008), cálculo de backoff (ADR-002), geração e verificação de assinatura HMAC (ADR-003), e das novas classes de erro `WEBHOOK_*`.
- **Testes de integração**: fluxo completo de `changeStatus` gerando (ou não, conforme filtro) uma linha em `webhook_outbox` dentro da mesma transação, seguindo o padrão já usado em `tests/orders.test.ts`.
- **Testes do worker**: simulação de endpoint de cliente respondendo com sucesso, timeout, erro 5xx e erro 4xx, verificando a transição correta de estados (`pending` → `delivered` ou `pending` → nova tentativa agendada → `dead_letter`).
- **Revisão de segurança dedicada**: validação manual do fluxo de geração/rotação de secret e da verificação de HMAC pela engenheira de segurança antes do deploy em produção, conforme compromisso assumido na reunião de definição.
