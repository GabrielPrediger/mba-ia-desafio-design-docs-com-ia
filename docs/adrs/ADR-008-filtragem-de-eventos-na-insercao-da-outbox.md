# ADR-008: Filtragem de eventos por status de interesse na inserção da outbox

## Status

Aceito

## Contexto

Cada webhook cadastrado por um cliente pode especificar uma lista de status de pedido que deseja ouvir (ex.: "só quero saber quando virar `SHIPPED` e `DELIVERED`"). É preciso decidir em que ponto do fluxo esse filtro é aplicado: no momento em que o evento entraria na `webhook_outbox` (antes de persistir), ou apenas no momento do envio pelo worker (persistindo todos os eventos e descartando no processamento os que não interessam a nenhum webhook).

## Decisão

O filtro é aplicado **no momento da inserção na outbox**: ao processar uma mudança de status dentro de `OrderService.changeStatus`, a função de publicação (`publishWebhookEvent`, ver FDD) consulta quais webhooks ativos do `customer_id` do pedido têm aquele `to_status` na lista de interesse, e insere uma linha na `webhook_outbox` **apenas** para esses webhooks. Se nenhum webhook cadastrado do cliente quiser aquele status, nenhuma linha é inserida.

## Alternativas Consideradas

- **Persistir o evento para todos os webhooks do cliente e filtrar apenas no momento do envio**: descartada. Geraria linhas na tabela de outbox que nunca seriam de fato enviadas, desperdiçando espaço e aumentando o volume que o worker precisa varrer a cada ciclo de polling, sem nenhum benefício correspondente.

## Consequências

**Positivas:**
- Tabela de outbox contém apenas eventos que de fato serão entregues, mantendo-a menor e mais barata de consultar pelo worker.
- Lógica de "quem recebe o quê" fica centralizada em um único ponto (a função de publicação), mais fácil de auditar.

**Negativas:**
- Se um webhook for cadastrado ou tiver seus filtros de evento alterados **depois** de uma mudança de status já ter ocorrido, não há como "voltar no tempo" e gerar retroativamente o evento perdido — comportamento esperado e aceito, mas que vale estar documentado (ver FDD, seção de riscos).

## Rastreabilidade

Transcrição: [09:33]–[09:34] (Marcos, Diego, Bruno).
