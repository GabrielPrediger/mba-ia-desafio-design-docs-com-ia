# ADR-002: Retry com backoff exponencial e Dead Letter Queue (DLQ) persistida

## Status

Aceito

## Contexto

Endpoints de clientes externos podem estar temporariamente indisponíveis (manutenção, instabilidade de rede, deploys). O time já tinha um caso real de cliente com indisponibilidade de cerca de duas horas durante uma janela de manutenção planejada. O sistema de webhooks precisa tolerar essas janelas sem perder o evento e sem martelar o cliente indefinidamente.

Duas dimensões precisavam de decisão: quantas tentativas fazer antes de desistir, e o que fazer com um evento que esgotou as tentativas.

## Decisão

Adotar **retry com backoff exponencial, 5 tentativas**, com a progressão de intervalos **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** entre tentativas (cobrindo uma janela de aproximadamente 15 horas entre a primeira falha e a última tentativa). Esse número cobre confortavelmente janelas de indisponibilidade planejada observadas na prática, sem deixar o evento pendurado indefinidamente.

Quando as 5 tentativas se esgotam, o evento é movido para uma tabela separada, `webhook_dead_letter`, contendo a payload original, o motivo da última falha e o timestamp. Manter a DLQ em tabela própria (em vez de apenas marcar `failed` na própria `webhook_outbox`) mantém a tabela principal de outbox limpa para o worker operar, e dá uma trilha de evidência dedicada para debug e reprocessamento.

O reprocessamento de um item da DLQ é manual, via endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na `webhook_outbox` com status pendente (ver ADR-006 sobre a exigência de role ADMIN nesse endpoint).

## Alternativas Consideradas

- **3 tentativas**: descartada por ser agressiva demais — uma indisponibilidade de cliente de algumas dezenas de minutos (cenário já observado) já seria suficiente para esgotar as tentativas e perder a entrega automática do evento.
- **Retry indefinido sem teto**: descartada porque deixaria eventos presos indefinidamente para clientes que saíram do ar permanentemente, sem um ponto de decisão claro de "isso precisa de intervenção humana".
- **Marcar falha definitiva diretamente na tabela de outbox (`status = failed`)** em vez de tabela de DLQ separada: descartada por poluir a tabela operacional principal com registros que não fazem mais parte do fluxo ativo do worker, dificultando tanto a leitura de pendentes quanto a investigação de falhas.

## Consequências

**Positivas:**
- Cliente com indisponibilidade temporária (até ~15h) ainda recebe o evento automaticamente, sem intervenção humana.
- Falhas permanentes ficam isoladas e auditáveis em `webhook_dead_letter`, com motivo e payload preservados para diagnóstico.
- Reprocessamento manual e controlado (via endpoint ADMIN) evita reenvio acidental ou em massa sem supervisão.

**Negativas:**
- Um evento pode demorar até ~15h para ser definitivamente marcado como falho, o que é aceitável para o caso de uso mas deve ser comunicado claramente aos clientes (responsabilidade do time de produto).
- Reprocessamento manual implica que, se ninguém monitorar a DLQ, eventos falhos podem ficar sem replay por tempo indefinido — não há alerta automático nesta fase (fora de escopo; ver PRD, seção "Fora de escopo").

## Rastreabilidade

Transcrição: [09:14]–[09:19] (Diego, Bruno, Larissa).
