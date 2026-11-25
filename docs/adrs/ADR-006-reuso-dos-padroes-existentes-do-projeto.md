# ADR-006: Reuso máximo dos padrões arquiteturais já existentes no projeto

## Status

Aceito

## Contexto

O Order Management System já possui convenções estabelecidas e consistentes entre todos os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`): estrutura de módulo com `controller` + `service` + `repository` + `routes` + `schemas`; tratamento de erros via `AppError` e subclasses com `errorCode` (ex.: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) capturadas centralmente pelo `errorMiddleware`; logging estruturado via Pino; autenticação JWT com `authenticate`/`requireRole`; validação de entrada com Zod. A pergunta levantada na reunião foi se o novo módulo de webhooks deveria introduzir padrões próprios ou seguir exatamente o que já existe.

## Decisão

O módulo de webhooks **reaproveita integralmente** os padrões arquiteturais já estabelecidos, sem introduzir convenções novas:

- **Estrutura de módulo**: `src/modules/webhooks/` seguindo o mesmo layout de `controller.ts` + `service.ts` + `repository.ts` + `routes.ts` + `schemas.ts` usado por `src/modules/orders/*` e os demais módulos.
- **Erros**: uma família de erros derivada de `AppError` (`src/shared/errors/app-error.ts`) com códigos prefixados por **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo molde de `InsufficientStockError`/`InvalidStatusTransitionError` em `src/shared/errors/http-errors.ts`. O `errorMiddleware` (`src/middlewares/error.middleware.ts`) já trata qualquer `AppError`, `ZodError` e erros conhecidos do Prisma sem nenhuma alteração necessária.
- **Logging**: reuso do logger Pino já configurado em `src/shared/logger/index.ts`, sem introduzir nova dependência de observabilidade.
- **Autenticação/autorização**: reuso de `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts` — o endpoint de replay de DLQ exige `requireRole('ADMIN')`, os demais endpoints do CRUD de configuração exigem apenas autenticação (qualquer role), podendo ser restringido no futuro.
- **Validação**: schemas Zod no mesmo estilo de `src/modules/orders/order.schemas.ts` (ex.: `z.string().url()` para a URL do webhook, rejeitando `http://` por exigir `https://`).
- **Identificadores**: UUID como chave primária em todas as novas tabelas (`webhook_outbox`, `webhook_dead_letter`, e a tabela de configuração de webhooks), consistente com o padrão `@default(uuid()) @db.Char(36)` usado em todas as entidades existentes no `prisma/schema.prisma` (`User`, `Customer`, `Order`, etc.).
- **Infraestrutura de banco**: mesma instância de banco MySQL via Prisma, nenhuma tecnologia nova introduzida pela feature em si (ver ADR-001).

## Alternativas Consideradas

- **Criar um padrão de erros ou de logging próprio para o módulo de webhooks** (por ser um domínio "diferente", voltado a integrações externas): descartada. Fragmentaria a consistência da base de código sem nenhum ganho real — o padrão existente já cobre as necessidades do novo módulo.
- **ID autoincremental para as novas tabelas** (mais simples/comum em alguns projetos): descartada em favor de UUID, para manter uniformidade com o restante do schema (`prisma/schema.prisma`), evitando uma exceção não justificada.

## Consequências

**Positivas:**
- Qualquer desenvolvedor já familiarizado com os módulos existentes (`orders`, `products`, etc.) consegue navegar o módulo de webhooks sem curva de aprendizado adicional.
- Zero código de infraestrutura novo (middleware de erro, logger, autenticação) — reduz superfície de bugs e retrabalho.
- Facilita revisão de código e futuras manutenções, por manter um único "jeito de fazer as coisas" no projeto.

**Negativas:**
- Caso algum padrão existente se mostre inadequado especificamente para webhooks (ex.: necessidade futura de rate limiting, não contemplada hoje em nenhum módulo), será necessário estender o padrão para todo o projeto ou abrir uma exceção documentada — nenhum dos dois caminhos foi necessário nesta fase.

## Rastreabilidade

Transcrição: [09:27]–[09:30] (Bruno, Larissa, Diego). Código: `src/modules/orders/*` (estrutura de módulo de referência), `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`.
