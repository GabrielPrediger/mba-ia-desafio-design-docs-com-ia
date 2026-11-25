# ADR-005: Worker de entrega como processo separado, em polling de 2 segundos

## Status

Aceito

## Contexto

Com o padrão outbox decidido (ADR-001), é preciso definir quem lê a tabela `webhook_outbox` e dispara as entregas, e como esse componente se relaciona com o processo da API existente (`src/server.ts`).

Rodar a lógica de entrega dentro do mesmo processo da API criaria acoplamento de ciclo de vida: se a API reiniciar (deploy, crash, scaling), o worker reinicia junto, mesmo que não houvesse motivo para interromper o processamento de eventos pendentes. Também foi discutido como o worker deveria "descobrir" novos eventos: MySQL não oferece um mecanismo nativo de notificação para processos externos (ao contrário do `LISTEN`/`NOTIFY` do PostgreSQL) — triggers de banco só executam SQL, não conseguem acordar um processo Node externo sem soluções improvisadas (escrever em arquivo, chamar um endpoint).

## Decisão

Implementar o worker como um **processo Node separado**, com entry-point próprio (`src/worker.ts`, análogo ao `src/server.ts` existente, exposto via um script `npm run worker` em `package.json`), que se conecta ao **mesmo banco de dados** (`DATABASE_URL`) usando uma **instância própria de `PrismaClient`** (não compartilhada com o processo da API — `PrismaClient` é por processo, seguindo o mesmo padrão de inicialização de `src/config/database.ts`).

O worker opera em **polling com intervalo de 2 segundos**: a cada ciclo, busca um lote pequeno dos eventos pendentes mais antigos (ordenados por `created_at`), processa e marca como entregue ou agenda retry. Um polling de 2 segundos atende folgadamente o requisito funcional de notificação "abaixo de 10 segundos" definido pelos clientes B2B, sem exigir nenhuma infraestrutura de notificação adicional.

**Limitação de ordering documentada**: com um único worker processando em ordem de `created_at`, a ordem de entrega para um mesmo pedido é preservada (ex.: `PAID` → `PROCESSING` → `SHIPPED` chegam na ordem certa). Essa garantia é implícita por `order_id` e depende de haver exatamente um worker ativo. Não há garantia de ordering global entre pedidos diferentes, e se o sistema escalar para múltiplos workers em paralelo no futuro, a garantia de ordering por pedido também se perde a menos que se introduza particionamento por `order_id` ou lock pessimista — ambos fora do escopo desta fase.

## Alternativas Consideradas

- **Worker embutido no mesmo processo da API** (ex.: um `setInterval` dentro de `src/server.ts`): descartada porque acopla o ciclo de vida do processamento de eventos ao ciclo de vida de deploy/restart da API, podendo interromper ou atrasar entregas sem necessidade.
- **Reação via trigger de banco de dados**: descartada porque MySQL não tem mecanismo nativo para um trigger notificar um processo externo (diferente do Postgres `LISTEN`/`NOTIFY`); as alternativas improvisadas (grafar em arquivo, chamar endpoint a partir do banco) foram consideradas inadequadas ("fica esquisito").
- **Múltiplos workers em paralelo desde o início**: não adotada nesta fase. Resolveria throughput em alta escala, mas introduziria o problema de ordering sem necessidade imediata, dado que o volume atual não demanda isso — tratado como melhoria futura documentada como limitação conhecida.

## Consequências

**Positivas:**
- Processamento de eventos sobrevive a reinícios/deploys da API, pois roda em processo independente.
- Simplicidade operacional: nenhuma infraestrutura de mensageria nova, apenas um segundo entry-point Node conectado ao mesmo MySQL.
- Atende o requisito de latência ("abaixo de 10s") com folga usando um mecanismo simples (polling).

**Negativas:**
- Introduz uma latência mínima de até 2 segundos por natureza do polling (aceita como trade-off).
- Ordering só é garantida por `order_id` e apenas em configuração single-worker; escalar horizontalmente exigirá trabalho adicional não coberto por esta decisão.
- Um segundo processo para operar, monitorar e fazer deploy (ainda que compartilhando a mesma base de código e banco).

## Rastreabilidade

Transcrição: [09:03]–[09:13] (Larissa, Bruno, Diego). Código: `src/server.ts` (padrão de entry-point/bootstrap a ser replicado em `src/worker.ts`), `src/config/database.ts` (padrão de criação do `PrismaClient`), `package.json` (seção `scripts`).
