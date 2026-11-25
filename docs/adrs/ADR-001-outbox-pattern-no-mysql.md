# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito

## Contexto

O Order Management System precisa notificar sistemas externos (clientes B2B: Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda. A mudança de status já ocorre dentro de uma transação que atualiza `orders`, insere em `order_status_history` e, dependendo da transição, debita ou repõe `stock_quantity` em `products` (ver `src/modules/orders/order.service.ts`, método `changeStatus`).

Duas alternativas de disparo foram discutidas: (1) chamar o endpoint do cliente de forma síncrona, dentro da própria transação de mudança de status, ou (2) registrar o evento de forma transacional e processá-lo de forma assíncrona por um worker separado.

Chamar o cliente de forma síncrona significa fazer uma chamada HTTP para um sistema fora do nosso controle (e potencialmente lento ou indisponível) no meio de uma transação de banco que já é pesada. Se o cliente estiver fora do ar, não há como simplesmente abortar a mudança de status (dar rollback) — o pedido já mudou de estado de verdade. Isso tornaria a confiabilidade da nossa operação interna refém da disponibilidade de terceiros.

## Decisão

Adotar o padrão **Transactional Outbox** sobre o MySQL já existente (sem introduzir nova infraestrutura): dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir também uma linha em uma nova tabela `webhook_outbox` contendo o evento já renderizado (ver ADR-007 sobre snapshot). Se a transação principal falhar e sofrer rollback, o evento nunca existiu. Se ela commitar, o evento está garantidamente persistido. Um worker separado (ver ADR-005) lê a tabela de forma assíncrona e dispara as chamadas HTTP, sem acoplar a latência ou disponibilidade do cliente à transação de domínio.

A tabela `webhook_outbox` terá índice composto em `status` (pendente, processando, falhou, entregue) e `created_at`, para que o worker leia eficientemente apenas os pendentes mais antigos em lotes pequenos. Linhas entregues podem ser arquivadas após um período de retenção (ex.: 30 dias); essa rotina de arquivamento fica fora do escopo desta feature.

## Alternativas Consideradas

- **Disparo síncrono no `OrderService.changeStatus`**: descartada. Acoplaria a confiabilidade da mudança de status (uma operação interna crítica) à disponibilidade e latência de um sistema de terceiros, e não há como fazer rollback de um pedido que já mudou de estado de fato.
- **Fila dedicada (ex.: Redis Streams/RabbitMQ)**: descartada para este momento. Resolveria o mesmo problema, mas exigiria subir e operar infraestrutura nova para um time pequeno — overengineering frente ao volume e à urgência do requisito ("abaixo de 10 segundos" já é atendido por polling simples, ver ADR-005). Outbox reaproveitando o MySQL já existente resolve com custo operacional zero adicional.

## Consequências

**Positivas:**
- Garante atomicidade entre a mudança de estado do pedido e o registro do evento — nunca existe mudança de status sem evento correspondente, nem evento "órfão" de uma mudança que não ocorreu.
- Não introduz nova peça de infraestrutura (banco, broker) para o time operar e monitorar.
- Modelo simples de entender e depurar: o estado de cada evento é uma linha de tabela relacional, consultável via SQL comum.

**Negativas:**
- O worker precisa fazer polling (sem push nativo do MySQL), o que introduz uma latência mínima inerente ao intervalo de polling (ver ADR-005) — aceito porque ainda atende folgadamente o requisito de "abaixo de 10 segundos".
- Em alto volume, a tabela de outbox cresce continuamente e precisa de uma estratégia de arquivamento/retenção — deixado como melhoria futura, fora do escopo desta fase.
- Escalar para múltiplos workers em paralelo no futuro exigirá mecanismo adicional de particionamento ou lock (ver limitação de ordering na ADR-005), não resolvido nesta decisão.

## Rastreabilidade

Transcrição: [09:03]–[09:08] (Larissa, Bruno, Diego). Código: `src/modules/orders/order.service.ts` (método `changeStatus`).
