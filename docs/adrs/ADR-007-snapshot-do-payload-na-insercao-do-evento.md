# ADR-007: Snapshot do payload no momento da inserção do evento na outbox

## Status

Aceito

## Contexto

Ao inserir um evento na `webhook_outbox`, existem duas formas de guardar a informação do que será enviado: (a) guardar apenas o `order_id` e renderizar o payload completo no momento do envio (quando o worker processa a linha), ou (b) já renderizar e persistir o payload completo (JSON) no momento da inserção, como um snapshot do estado do pedido naquele instante.

Como o worker processa de forma assíncrona (polling de 2 segundos, ADR-005) e pode haver retries ao longo de várias horas (ADR-002), o pedido pode sofrer novas mudanças de estado entre o momento em que o evento foi gerado e o momento em que efetivamente é enviado (ou reenviado).

## Decisão

O payload é **renderizado e persistido como snapshot no momento da inserção** do evento na `webhook_outbox`, refletindo o estado do pedido exatamente no instante da transição de status que originou o evento (`from_status` → `to_status`, dados básicos do pedido como `total_cents`, etc. — ver FDD para o contrato completo). O worker apenas lê e envia o payload já pronto, sem recalculá-lo a partir do estado atual do pedido.

## Alternativas Consideradas

- **Guardar apenas `order_id` e renderizar o payload no momento do envio**: descartada. Se o pedido mudasse de estado novamente antes do envio (ou durante uma janela de retry de horas), o evento enviado refletiria um estado diferente daquele que efetivamente motivou a notificação — o cliente poderia, por exemplo, receber um evento de "mudou para `PROCESSING`" mas com o payload já mostrando `SHIPPED`, uma inconsistência confusa e potencialmente perigosa para quem automatiza decisões a partir do webhook.

## Consequências

**Positivas:**
- Cada evento entregue reflete fielmente o estado do pedido no momento exato da transição que o originou, mesmo que o pedido tenha avançado mais desde então.
- Elimina uma classe inteira de bugs de "estado inconsistente entre o que o evento diz e o que o banco diz agora".

**Negativas:**
- Pequeno aumento de armazenamento por linha da outbox (payload completo em vez de apenas uma referência), considerado desprezível frente ao limite de 64KB por payload (ver FDD).
- Se houver necessidade futura de enriquecer eventos antigos com dados calculados posteriormente, isso exigiria reprocessamento manual, não um simples "re-render" automático.

## Rastreabilidade

Transcrição: [09:51]–[09:53] (Bruno, Larissa, Diego).
