# Da Reunião ao Documento: Design Docs Gerados por IA

> O enunciado original deste desafio está preservado na plataforma do curso (MBA em Engenharia de Software com IA — Full Cycle, desafio "Design Docs com IA"). Este README documenta o processo de produção da entrega, conforme pedido no requisito 6 do enunciado.

## Sobre o Desafio

O desafio pede para transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) sobre um novo Sistema de Webhooks de Notificação de Pedidos em um pacote completo de design docs — PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade — usando IA como ferramenta principal de produção, sem alterar o código da aplicação existente (um Order Management System em Node.js/TypeScript/Prisma/MySQL). O ponto central da avaliação não é só gerar documentos bonitos, mas garantir que cada requisito, decisão e restrição registrada seja rastreável à transcrição real ou ao código real — nada pode ser inventado.

## Ferramentas de IA Utilizadas

- **Claude (via Claude Code / Cowork)**: ferramenta principal de produção. Foi usada para ler integralmente `TRANSCRICAO.md` e os arquivos-fonte relevantes do repositório base, extrair as decisões e requisitos discutidos na reunião, e redigir os seis documentos (PRD, RFC, FDD, os 8 ADRs, o Tracker e este README) a partir desse material, seguindo a ordem de execução sugerida no enunciado (ADRs → RFC → FDD → PRD → Tracker → README).
- **Claude in Chrome (automação de navegador)**: usada apenas para a etapa de fork do repositório base no GitHub (clicar em "Fork" autenticado como o próprio usuário) — não participou da produção de conteúdo dos documentos.

## Workflow Adotado

1. **Leitura completa do material-fonte**: a transcrição inteira (~55 minutos, 5 participantes) e os arquivos de código mais relevantes para a integração (`order.service.ts`, `order.status.ts`, `app-error.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `schema.prisma`, entre outros) foram lidos antes de qualquer documento ser escrito.
2. **ADRs primeiro**: seguindo a recomendação do próprio enunciado, as decisões arquiteturais foram identificadas e documentadas antes dos demais documentos, já que formam o "esqueleto" do RFC e do FDD. Isso resultou em 6 ADRs cobrindo as decisões principais listadas no enunciado, mais 2 ADRs adicionais para decisões secundárias que se mostraram relevantes o suficiente para merecer registro isolado (momento de snapshot do payload e ponto de aplicação do filtro de eventos).
3. **RFC**: consolidado em cima dos ADRs já prontos, mantendo-o deliberadamente conciso (nível de arquitetura, não de implementação) e linkando cada decisão fechada ao ADR correspondente.
4. **FDD**: escrito depois, aprofundando em fluxos, contratos HTTP e a seção obrigatória de integração com o código existente, sem repetir o que já estava decidido e justificado no RFC/ADRs.
5. **PRD**: produzido por último entre os grandes documentos, como uma consolidação de nível de produto em cima do que já estava formalizado tecnicamente.
6. **Tracker**: montado ao final, varrendo PRD, RFC, FDD e ADRs linha a linha para mapear cada item à origem (timestamp da transcrição ou caminho de arquivo).
7. **Este README**: escrito por último, descrevendo o processo já concluído.

## Prompts Customizados

Dois exemplos representativos da forma como a IA foi direcionada para cada tipo de documento (em vez de pedidos genéricos como "gere um PRD a partir dessa transcrição"):

Para os ADRs:

```
Leia TRANSCRICAO.md e extraia as 6 decisões arquiteturais principais
discutidas (outbox no MySQL, retry/backoff/DLQ, HMAC-SHA256 por
endpoint, at-least-once com X-Event-Id, worker separado em polling,
reuso de padrões existentes). Para cada uma, produza um ADR em
formato MADR (Status, Contexto, Decisão, Alternativas Consideradas,
Consequências) em arquivo próprio dentro de docs/adrs/, nomeado
ADR-NNN-titulo-em-kebab-case.md. Cada ADR deve citar o timestamp e o
falante exatos da transcrição que embasam a decisão, e pelo menos um
ADR deve referenciar um arquivo real do código-base (ex.: o método
changeStatus em order.service.ts). Não infira nenhuma decisão que não
tenha sido explicitamente fechada na reunião.
```

Para o FDD, com foco na seção de integração exigida pelo enunciado:

```
Com base nos ADRs já escritos e no código em src/modules/orders/ e
src/shared/errors/, escreva a seção "Integração com o sistema
existente" do FDD. Cite pelo menos 4 caminhos de arquivo reais do
repositório e descreva, para cada um, exatamente como o módulo de
webhooks vai se acoplar a ele — por exemplo, qual método de
order.service.ts é estendido e com qual chamada, e quais classes de
erro existentes servem de molde para as novas classes WEBHOOK_*. Não
descreva integração com nenhum arquivo que não exista de fato no
repositório.
```

## Iterações e Ajustes

O processo não foi gerar tudo de uma vez: alguns pontos exigiram correção explícita durante a produção.

1. **Fronteira RFC x FDD**: a primeira passagem mental sobre o RFC tendia a repetir detalhe de implementação (contratos HTTP completos, matriz de erros) que o próprio enunciado reserva ao FDD. Foi necessário reforçar ativamente a regra "o RFC propõe e abre para revisão, o FDD detalha como construir" para manter o RFC em 2-4 páginas conceituais, deixando contratos e matriz de erros inteiramente para o FDD.
2. **Contagem de ADRs**: a varredura inicial da transcrição apontava claramente as 6 decisões obrigatórias listadas no enunciado. Numa segunda leitura mais atenta da parte final da reunião (conversa informal entre Diego, Larissa e Bruno após a saída de Marcos e Sofia, por volta de [09:51]–[09:53]), identificaram-se duas decisões técnicas reais adicionais (UUID vs. autoincremental para a outbox, e snapshot do payload na inserção vs. renderização tardia) que mereciam registro próprio em vez de ficarem perdidas só no corpo do FDD — resultando em 8 ADRs ao todo, dentro do limite de 5 a 8 permitido.
3. **Cobertura do Tracker**: a primeira versão do tracker citava quase exclusivamente a transcrição como fonte. Para cumprir o critério de pelo menos 5 linhas com `Fonte = CODIGO` e manter a tabela como uma ferramenta real de verificação (e não só de citação de falas), várias linhas foram revisadas para apontar diretamente o arquivo de código que efetivamente ancora aquele item do FDD (ex.: o padrão de classes de erro em `http-errors.ts`, o middleware de erro centralizado, o schema do Prisma), em vez de apenas repetir a justificativa da reunião.

## Como Navegar a Entrega

Ordem de leitura sugerida, do mais estratégico ao mais técnico:

1. [`docs/PRD.md`](docs/PRD.md) — problema, público, escopo e critérios de sucesso.
2. [`docs/RFC.md`](docs/RFC.md) — proposta técnica de alto nível, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 8 decisões arquiteturais individuais (ADR-001 a ADR-008), cada uma com seu próprio contexto e trade-offs.
4. [`docs/FDD.md`](docs/FDD.md) — especificação de implementação: fluxos, contratos HTTP, matriz de erros, observabilidade e a seção de integração com o código existente.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — tabela de rastreabilidade cruzando todo o conteúdo acima com `TRANSCRICAO.md` e o código-fonte.

O código da aplicação (`src/`, `prisma/`, `tests/`) e a transcrição original (`TRANSCRICAO.md`) permanecem inalterados, servindo apenas como contexto e referência, conforme exigido pelo enunciado.
