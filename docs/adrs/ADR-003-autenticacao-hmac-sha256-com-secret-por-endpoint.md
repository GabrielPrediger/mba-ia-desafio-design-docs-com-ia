# ADR-003: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

Os eventos de webhook carregam dados de pedidos de clientes reais e são enviados para endpoints fora da nossa infraestrutura. Do lado do cliente, é preciso haver uma forma de verificar que (a) a requisição realmente partiu de nós e (b) o payload não foi adulterado em trânsito. Sem isso, qualquer terceiro que descobrisse a URL do endpoint do cliente poderia forjar notificações de mudança de pedido.

Também foi levantado um risco concreto: o time já teve um incidente em que um cliente vazou, em log de aplicação dele, a secret associada ao webhook — o que reforça a necessidade de a secret ser rotacionável sem causar downtime na validação do lado do cliente.

## Decisão

Assinar o corpo de cada requisição de webhook com **HMAC-SHA256**, usando uma **secret única por endpoint cadastrado** (nunca uma secret global da plataforma), e enviar a assinatura resultante no header `X-Signature`. A secret é gerada pela nossa API no momento do cadastro do webhook e devolvida uma única vez na resposta de criação.

A secret é **rotacionável** via endpoint dedicado: ao rotacionar, a secret antiga permanece válida em paralelo por um **grace period de 24 horas**, permitindo que o cliente atualize seus sistemas sem uma janela de falha de validação. Após as 24 horas, apenas a nova secret é aceita.

HMAC-SHA256 foi escolhido por ser o padrão de mercado para este tipo de verificação (amplamente suportado por bibliotecas em qualquer stack do lado do cliente), equilibrando simplicidade de implementação dos dois lados com robustez criptográfica.

## Alternativas Consideradas

- **Secret global única para todos os clientes**: descartada. Se vazasse uma única vez (como já ocorreu na prática com um cliente), todos os outros clientes ficariam comprometidos simultaneamente. Secret por endpoint limita o raio de impacto de um vazamento a um único cliente/endpoint.
- **Rotação de secret sem grace period** (troca imediata, secret antiga invalidada na hora): descartada por criar uma janela garantida de falha de validação para o cliente até ele atualizar seus sistemas, o que é operacionalmente pior do que manter as duas secrets válidas por 24h.
- **Assinatura via JWT/assinatura assimétrica (RSA/ECDSA)**: não chegou a ser proposta na reunião como alternativa séria; HMAC simétrico foi adotado diretamente por ser mais simples de implementar nos dois lados e suficiente para o modelo de ameaça (par de segredos compartilhados entre duas partes já confiáveis uma na outra, não um sistema de verificação pública).

## Consequências

**Positivas:**
- Cliente consegue validar autenticidade e integridade de cada evento recebido sem depender de uma conexão de volta à nossa API.
- Vazamento de uma secret compromete apenas um endpoint/cliente, não a plataforma inteira.
- Rotação com grace period permite resposta a incidentes de vazamento sem gerar indisponibilidade imediata para o cliente afetado.

**Negativas:**
- Exige que o cliente implemente corretamente a verificação HMAC do lado dele; erros de implementação no cliente (ex.: comparação não constante-no-tempo) estão fora do nosso controle.
- Manter duas secrets válidas simultaneamente durante o grace period aumenta levemente a superfície de validação (o worker/validador precisa testar contra ambas durante a janela).

## Rastreabilidade

Transcrição: [09:19]–[09:22] (Sofia, Bruno, Diego).
