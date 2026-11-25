# ADR-004: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceito

## Contexto

Dado o modelo de retry com backoff (ADR-002), existe a possibilidade real de um cliente receber o mesmo evento mais de uma vez — por exemplo, se a chamada HTTP foi processada com sucesso do lado do cliente mas a resposta se perdeu antes de chegar ao worker, que então trata como falha e reenvia. O time precisava decidir qual garantia de entrega oferecer e como o cliente deveria lidar com duplicatas.

## Decisão

Garantir **at-least-once delivery**: o evento é entregue pelo menos uma vez, podendo ocasionalmente ser entregue mais de uma vez. Para permitir que o cliente deduplique do lado dele, cada evento carrega um identificador único (UUID) gerado no momento em que entra na `webhook_outbox`, enviado no header **`X-Event-Id`**. O cliente é responsável por usar esse identificador para detectar e ignorar entregas duplicadas.

Essa responsabilidade é explicitamente transferida para o cliente, documentada de forma destacada no portal de desenvolvedor, seguindo o mesmo padrão adotado por provedores de referência do mercado (ex.: Stripe, GitHub) para webhooks.

## Alternativas Consideradas

- **Exactly-once delivery**: descartada. Exigiria coordenação transacional entre os dois lados (nosso sistema e o sistema do cliente), o que é significativamente mais complexo de implementar e operar, e está fora do controle unilateral da nossa plataforma — não é possível garantir exactly-once sem cooperação ativa do destinatário.
- **At-most-once (entregar no máximo uma vez, sem retry em caso de dúvida)**: descartada implicitamente ao se adotar retry com backoff (ADR-002); priorizar "nunca duplicar" teria exigido desistir de reentregas em cenários ambíguos, arriscando perda silenciosa de eventos — inaceitável para um sistema de notificação de mudança de pedido.

## Consequências

**Positivas:**
- Resolve a esmagadora maioria dos casos práticos (entrega confiável, sem exigir infraestrutura de coordenação distribuída).
- Simples de implementar em ambos os lados: gerar um UUID na inserção do evento, enviar no header, e o cliente mantém um conjunto/cache de IDs já processados.

**Negativas:**
- Transfere trabalho de implementação para o cliente (ele precisa deduplicar), o que precisa ser comunicado com clareza no onboarding/portal do desenvolvedor para evitar clientes tratando duplicatas como pedidos novos.
- Não há, do lado da nossa plataforma, nenhuma garantia adicional além do header — se o cliente não implementar a deduplicação, duplicatas terão efeito real do lado dele.

## Rastreabilidade

Transcrição: [09:24]–[09:26] (Diego, Bruno, Sofia, Marcos).
