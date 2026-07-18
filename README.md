# Da Reunião ao Documento — Design Docs Gerados por IA

Entrega do desafio de produção de design docs com IA a partir da transcrição de uma reunião técnica e do código de um OMS existente.

> O enunciado original do desafio pode ser consultado no histórico do git deste repositório (README anterior a esta entrega) ou no repositório base do curso.

## Sobre o desafio

Uma reunião técnica de ~55 minutos ([TRANSCRICAO.md](./TRANSCRICAO.md)) decidiu como um Order Management System em produção vai ganhar um **Sistema de Webhooks de Notificação de Pedidos** — mas nada foi registrado além da transcrição. O desafio é transformar essa transcrição, junto com o código existente, em um pacote completo de design docs (PRD, RFC, FDD, ADRs e um tracker de rastreabilidade), em nível acionável o suficiente para o time começar a implementar.

A regra central é a rastreabilidade: nenhum requisito, decisão ou restrição pode ser inventado — tudo precisa ter origem identificável na transcrição (timestamp + falante) ou no código (caminho de arquivo). O papel humano aqui é o de "maestro": dirigir a IA com prompts específicos, revisar criticamente o que ela produz e garantir que o resultado seja consistente entre os documentos, sem duplicação de conteúdo entre alturas diferentes (produto → arquitetura → decisão → implementação).

## Ferramentas de IA utilizadas

- **Claude Code (modelo Claude Fable 5, Anthropic):** ferramenta principal, usada de ponta a ponta. Por rodar como agente no terminal com acesso ao repositório, leu a transcrição e o código-fonte diretamente (em vez de depender de colagem manual de contexto), extraiu decisões e requisitos, gerou os documentos em Markdown e validou referências cruzadas (caminhos de arquivo citados, links entre documentos, contagens do tracker) com comandos de shell.

## Workflow adotado

O trabalho foi organizado em quatro etapas, na ordem em que os documentos dependem uns dos outros:

1. **Levantamento de contexto (antes de escrever qualquer documento):** leitura integral da `TRANSCRICAO.md` e dos arquivos-chave do código (`order.service.ts`, `order.status.ts`, `schema.prisma`, `app-error.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `logger`, `routes/index.ts`, `server.ts`, `package.json`). Isso garantiu que os documentos citassem apenas arquivos, classes e padrões que existem de fato.
2. **Extração dirigida da transcrição:** com prompts específicos (abaixo), a IA classificou cada fala em: decisão fechada, requisito funcional, requisito não funcional, alternativa descartada (com o trade-off), item adiado/fora de escopo e questão em aberto — cada item anotado com timestamp e falante. Essa lista virou a espinha dorsal de todos os documentos e, depois, do tracker.
3. **Produção em ordem de altura:** primeiro o **PRD** (por quê/o quê), depois os **6 ADRs** (uma decisão fechada por arquivo), então o **RFC** (proposta + alternativas + questões em aberto, já linkando os ADRs) e por fim o **FDD** (implementação em detalhe, incluindo a seção de integração com arquivos reais). Escrever os ADRs antes do RFC evitou duplicação: o RFC referencia, não repete.
4. **Tracker e verificação:** o **TRACKER.md** foi montado ao final, revisitando cada documento e mapeando cada item à sua origem. Em seguida, uma passada de verificação: conferência dos timestamps citados contra a transcrição, dos caminhos de arquivo contra o repositório (via `find`/`grep`) e das contagens de cobertura do tracker (recontadas por script, ver "Iterações").

## Prompts customizados

Prompt de extração dirigida (etapa 2) — o ponto importante é forçar a classificação e a origem, em vez de pedir um "resumo da reunião":

```text
Leia TRANSCRICAO.md na íntegra. Não resuma a reunião. Em vez disso, classifique cada
fala relevante em exatamente uma destas categorias:

1. DECISÃO FECHADA (houve consenso explícito ou a tech lead registrou como decidido)
2. REQUISITO FUNCIONAL (comportamento pedido pelo PM ou pelo time)
3. REQUISITO NÃO FUNCIONAL / RESTRIÇÃO (latência, limites, segurança, consistência)
4. ALTERNATIVA DESCARTADA (registre também o trade-off que motivou o descarte)
5. ADIADO / FORA DE ESCOPO (registre quem adiou e para quando)
6. QUESTÃO EM ABERTO (levantada e não decidida)

Para cada item, inclua obrigatoriamente o timestamp [hh:mm] e o nome do falante.
Atenção: nem tudo que foi mencionado é requisito — itens como email de falha, rate
limiting e dashboard foram explicitamente descartados ou adiados; eles devem aparecer
nas categorias 5/6, nunca como requisito.
```

Prompt de ancoragem no código (etapa 1/FDD) — usado para a seção "Integração com o sistema existente" e para o ADR de reuso:

```text
Antes de escrever a seção de integração do FDD, leia estes arquivos do repositório:
src/modules/orders/order.service.ts, src/modules/orders/order.status.ts,
src/shared/errors/app-error.ts, src/shared/errors/http-errors.ts,
src/middlewares/error.middleware.ts, src/middlewares/auth.middleware.ts,
src/shared/logger/index.ts, src/routes/index.ts, src/server.ts, prisma/schema.prisma.

Para cada ponto de integração citado no FDD: (a) use o caminho real do arquivo;
(b) cite o símbolo real (método, classe, função — ex.: changeStatus, AppError,
requireRole, TxClient) como ele aparece no código; (c) descreva a mudança em termos
do que já existe (ex.: "dentro do this.prisma.$transaction existente, após o
tx.orderStatusHistory.create"). É proibido citar arquivo, classe ou método que você
não tenha lido neste repositório.
```

Prompt de verificação do tracker (etapa 4):

```text
Para cada linha do TRACKER.md com Fonte = TRANSCRICAO, confirme que o timestamp
citado existe na TRANSCRICAO.md e que a fala daquele falante naquele minuto sustenta
o conteúdo resumido. Para cada linha com Fonte = CODIGO, confirme que o caminho
existe no repositório. Liste apenas as divergências.
```

## Iterações e ajustes

Principais correções feitas durante a produção (3 iterações principais até o resultado final):

1. **Autoria do `customer_id` no cadastro:** a primeira versão do contrato `POST /webhooks` assumia `customer_id` extraído do JWT — que é o desenho mais comum em APIs de webhook. A transcrição diz o contrário: Bruno aponta que o JWT é do **usuário operador**, não do cliente, e Larissa decide que o `customer_id` vai **no body ou path** ([09:32]). O contrato do FDD e o RF-01 do PRD foram corrigidos para refletir a decisão real da reunião.
2. **Códigos de erro além dos citados:** a matriz de erros inicial trazia códigos `WEBHOOK_*` adicionais (timeout, payload grande, replay duplicado) sem distinguir o que foi dito na reunião do que foi derivado. Como a reunião cita nominalmente apenas três códigos ("WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED, etc." — [09:28] Bruno), a matriz foi anotada para separar os códigos citados dos derivados do "etc." seguindo o padrão, e o tracker registra como origem apenas o padrão de prefixo — evitando apresentar invenção como decisão.
3. **Contagem de cobertura do tracker:** o resumo de cobertura gerado inicialmente afirmava totais que não batiam com a tabela real (94 linhas declaradas vs. 98 existentes). A contagem foi refeita por script (`grep -c` por fonte) e o resumo corrigido para os números verificados: 98 linhas, 84 com fonte na transcrição (~86%) e 14 no código — exemplo típico de "número plausível" de IA que só se sustenta se for medido, não estimado.
4. **Fronteira RFC × FDD:** a primeira estrutura do RFC repetia a progressão de retry e a lista de headers em detalhe. Esses níveis foram rebaixados para o FDD, deixando o RFC com a visão de arquitetura (diagrama, 6 pontos da proposta, alternativas e questões em aberto) e links para os ADRs — respeitando a regra de que conteúdo duplicado entre alturas é sinal de coisa no lugar errado.

## Como navegar a entrega

```
docs/
├── PRD.md          → por quê e o quê (produto, escopo, métricas, riscos)
├── RFC.md          → proposta técnica em revisão (alternativas, questões em aberto)
├── adrs/
│   ├── ADR-001-padrao-outbox-no-mysql.md
│   ├── ADR-002-worker-separado-com-polling.md
│   ├── ADR-003-retry-backoff-exponencial-dlq.md
│   ├── ADR-004-hmac-sha256-secret-por-endpoint.md
│   ├── ADR-005-at-least-once-x-event-id.md
│   └── ADR-006-reuso-padroes-existentes.md
├── FDD.md          → como construir (fluxos, contratos, erros, integração com o código)
└── TRACKER.md      → origem de cada item (transcrição ou código)
```

**Ordem sugerida de leitura:**

1. [TRANSCRICAO.md](./TRANSCRICAO.md) — a matéria-prima (opcional, mas ajuda a auditar).
2. [docs/PRD.md](./docs/PRD.md) — o problema, o escopo e o que ficou de fora.
3. [docs/RFC.md](./docs/RFC.md) — a proposta técnica e o que ainda está em aberto.
4. [docs/adrs/](./docs/adrs/) — cada decisão isolada, com alternativas e consequências (ADR-001 → ADR-006).
5. [docs/FDD.md](./docs/FDD.md) — a especificação de implementação.
6. [docs/TRACKER.md](./docs/TRACKER.md) — a auditoria: de onde veio cada coisa.

O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado — serve de contexto e referência para os documentos.
