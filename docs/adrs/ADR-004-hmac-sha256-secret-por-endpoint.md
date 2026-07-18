# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito — decisão fechada na reunião técnica ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").

## Contexto

Os webhooks expõem dados de pedidos para endpoints fora da nossa infraestrutura. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente de nós e que o payload não foi adulterado no caminho. Também há histórico real de incidente: um cliente já vazou uma secret em log da aplicação dele — o desenho precisa limitar o raio de dano de um vazamento e permitir recuperação sem downtime da integração.

## Decisão

- Assinar o **corpo do request** com **HMAC-SHA256**, usando uma secret compartilhada, enviada no header `X-Signature`. O cliente verifica do lado dele. HMAC-SHA256 é o padrão de mercado — todo cliente sério tem biblioteca para isso.
- **Secret única por endpoint** de webhook, não global da plataforma: "senão se vaza uma, vaza tudo". A configuração do webhook armazena `url + secret + customer_id + estado ativo`.
- A secret é **gerada pelo sistema** na criação do webhook e devolvida na resposta.
- **Rotação self-service**: endpoint para o cliente pedir nova secret pela API. Após a rotação, a secret antiga permanece **válida por 24 horas em paralelo** (grace period) para o cliente migrar os sistemas dele; depois disso, a antiga morre.
- Complementos de segurança da mesma discussão: URL obrigatoriamente `https` (validação no schema Zod, recusa com erro de validação) e header `X-Timestamp` no envio, permitindo ao cliente detectar replay attack se quiser.

## Alternativas Consideradas

1. **Secret global da plataforma** — descartado. Um único vazamento comprometeria a assinatura de todos os clientes ("se vaza uma, vaza tudo").
2. **Sem assinatura (confiar apenas no TLS)** — não aceitável para o cenário: TLS protege o transporte, mas não permite ao cliente autenticar a origem da requisição nem detectar adulteração por quem tenha acesso ao endpoint; a validação de origem foi colocada como requisito de segurança de partida ([09:19] Sofia).

## Consequências

**Positivas:**

- Cliente autentica origem e integridade do payload com verificação padrão de mercado.
- Vazamento de uma secret afeta um único endpoint de um único cliente; a rotação com grace de 24h permite recuperação sem quebrar a integração em produção.
- `X-Timestamp` dá ao cliente a opção de mitigar replay attacks.

**Negativas / trade-offs:**

- Durante o grace period de 24h, duas secrets são simultaneamente válidas para o endpoint — a verificação de assinatura na entrega precisa considerar ambas, e a janela de exposição de uma secret vazada se estende por até 24h após a rotação.
- Gestão de ciclo de vida de secrets (geração, rotação, expiração da antiga) adiciona estado e complexidade ao módulo.
- Exige revisão de segurança dedicada (Sofia reservou ≥ 2 dias úteis para revisar HMAC e geração de secret antes do deploy).
