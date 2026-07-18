# ADR-001 — Padrão Outbox no MySQL existente

## Status

Aceito — decisão fechada na reunião técnica ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

## Contexto

Os webhooks precisam ser disparados quando o status de um pedido muda. A transação de mudança de status (`changeStatus` em `src/modules/orders/order.service.ts`) já é pesada: atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos. A notificação precisa ser **consistente** com essa transação: não pode existir status alterado sem evento registrado, nem evento registrado com transação revertida. Além disso, o cliente notificado é um sistema externo, potencialmente lento ou fora do ar — sua indisponibilidade não pode afetar a operação de pedidos. O time é pequeno e quer evitar subir infraestrutura nova.

## Decisão

Adotar o **padrão outbox no MySQL já existente**: dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha em uma tabela `webhook_outbox` com o evento. Um worker separado lê essa tabela e dispara as chamadas HTTP.

Detalhes fechados na mesma discussão:

- A tabela tem índice no campo de status do evento (pendente, processando, falhou, entregue) e em `created_at`; o worker lê só os pendentes em batch pequeno.
- Chave primária **UUID**, seguindo o padrão de todo o projeto (`@default(uuid())` em `prisma/schema.prisma`).
- O evento guarda o **payload já renderizado (snapshot)** no momento da inserção: se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou.
- Arquivamento das linhas entregues (~30 dias) fica **fora do escopo** desta feature.

## Alternativas Consideradas

1. **Disparo síncrono dentro do service de orders** — descartado. Uma chamada HTTP no meio da transação faria qualquer cliente lento travar mudanças de status de outros pedidos, e não há resposta razoável para "cliente fora do ar" (rollback da mudança de status não é aceitável).
2. **Fila externa (Redis Streams ou similar)** — descartado. Exigiria subir e operar infraestrutura nova; para um time pequeno, "subir Redis Cluster pra isso é overengineering". O MySQL existente resolve o problema com garantia transacional nativa.

## Consequências

**Positivas:**

- Consistência garantida por construção: commit da transação principal ⇒ evento registrado; rollback ⇒ evento some junto. "Não tem inconsistência possível."
- Zero infraestrutura nova: mesmo MySQL, mesmo Prisma, mesma stack.
- A entrega HTTP fica completamente desacoplada da transação de pedidos.

**Negativas / trade-offs:**

- A entrega passa a depender de um worker próprio de polling (ver [ADR-002](./ADR-002-worker-separado-com-polling.md)), com a latência e a carga de leitura que isso implica.
- A tabela outbox cresce continuamente; sem o arquivamento (adiado), exige acompanhamento operacional.
- Acopla o mecanismo de eventos ao MySQL — uma eventual migração para fila dedicada no futuro exigirá refatoração do worker.
