# ADR-005 — Garantia at-least-once com deduplicação por X-Event-Id

## Status

Aceito — decisão fechada na reunião técnica ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão").

## Contexto

Com outbox + worker + retry, existe janela em que um evento pode ser entregue e a marcação de "entregue" falhar (crash do worker entre o POST e o update, timeout com o cliente tendo processado, retry após resposta perdida). É preciso definir a semântica de entrega prometida ao cliente: exactly-once ou at-least-once.

## Decisão

Garantir **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso.

Para permitir a deduplicação, todo envio carrega o header **`X-Event-Id`** com um **UUID gerado quando o evento entra na outbox** — único por evento e estável entre retries. Se o cliente receber duas vezes, deduplica pelo `event_id` do lado dele. O mesmo `event_id` também aparece no corpo do payload.

A semântica será documentada em destaque no portal de desenvolvedor para os clientes (compromisso do PM na reunião).

## Alternativas Consideradas

1. **Exactly-once** — descartado. Exigiria coordenação dos dois lados (nosso e do cliente) e ficaria muito mais complexo. "At-least-once com event_id resolve 99% dos casos."
2. **At-most-once (sem retry / fire-and-forget)** — incompatível com o requisito central da feature: o problema a resolver é justamente a confiabilidade da notificação (cliente offline não pode significar evento perdido), o que motivou toda a discussão de retry e DLQ ([09:14]–[09:17]).

## Consequências

**Positivas:**

- Modelo simples e robusto do nosso lado: retry sem medo de duplicar, sem protocolo de confirmação bilateral.
- É o padrão de mercado — "Stripe faz assim, GitHub faz assim" — então os clientes B2B já conhecem o modelo.
- `X-Event-Id` estável dá ao cliente uma chave de idempotência pronta.

**Negativas / trade-offs:**

- Joga responsabilidade para o cliente (apontado pela segurança na reunião): quem não implementar dedup pode processar o mesmo evento duas vezes.
- Exige documentação destacada no portal do desenvolvedor e alinhamento com os clientes piloto.
