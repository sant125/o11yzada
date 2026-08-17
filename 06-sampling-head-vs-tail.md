# Tema 6 · Sampling (head vs tail)

> `TL;DR:` Sampling é a válvula de custo de trace — guardar 100% das traces em alto volume é caro demais, e a solução é guardar uma amostra estatisticamente representativa. **Head sampling** decide no início, é barato e stateless, mas não enxerga o trace inteiro (não dá pra garantir "sempre guarda erro"). **Tail sampling** decide no final, depois de ver o trace completo, então consegue priorizar erro/latência — mas exige infra stateful pesada pra bufferizar traces inteiras antes de decidir.

---

## 1. A ideia antes do nome: amostra representa população, sem guardar tudo

> "Sampling is one of the most effective ways to reduce the costs of observability without losing visibility."

O princípio é estatístico, não uma gambiarra: uma amostra pequena e bem escolhida representa a população inteira. Em sistema de alto volume, taxa de **1% ou menos** já costuma ser suficiente pra enxergar padrão de latência, distribuição de erro, comportamento típico — sem pagar o custo de armazenar cada request individual. Isso já apareceu como conceito no Tema 4 (log também sampleia em volume alto), mas em trace é onde sampling é mais crítico, porque trace é o sinal mais caro por unidade (Tema 3, seção 1).

## 2. Head sampling: decide cedo, barato, mas cego ao resto do trace

Decisão tomada **no início** do trace — inspecionando só o que já se sabe naquele momento, não o trace completo (que ainda nem aconteceu).

A implementação mais comum é **Consistent Probability Sampling** (também chamada Deterministic Sampling): a decisão é baseada no **Trace ID** e numa taxa desejada — não é um sorteio novo a cada span, é uma função determinística do próprio ID.

Vantagens: "Easy to understand, easy to configure, efficient, can be done at any point" na pipeline — não precisa de componente stateful, roda em qualquer lugar (inclusive já no SDK, antes de qualquer coisa sair do processo).

Desvantagem que dói: **"It is not possible to make a sampling decision based on data in the entire trace"** — como a decisão é feita no início, é literalmente impossível garantir "sempre guarda trace com erro", porque o erro pode acontecer só num span filho, cinco saltos depois, quando a decisão de sampling já foi tomada.

## 3. Por que a decisão precisa ser consistente entre serviços — não um sorteio novo por hop

Antes de eu resolver: se cada serviço, ao receber uma request, jogasse sua própria moeda pra decidir "eu guardo esse span ou não" — independente do que o serviço anterior decidiu — o que aconteceria com uma trace que atravessa 8 serviços? Pensa no Tema 2: trace é uma **árvore**. O que sobra dessa árvore se cada nó decide, de forma independente e aleatória, se existe ou não?

### Resolução

Você acabaria com traces **fragmentadas** — span do serviço A presente, span do serviço C ausente (a moeda dele deu "não guarda"), span do serviço E presente de novo. Uma árvore com buracos no meio não é útil pra debugar nada; pior, pode até enganar (parece que a chamada pulou direto de A pra E). É por isso que head sampling usa uma função **determinística** do Trace ID, não um sorteio independente por serviço: todo serviço que recebe aquele mesmo Trace ID calcula a **mesma** decisão (ex: hash do trace ID caindo dentro do percentual configurado) — ou a árvore inteira é mantida, ou a árvore inteira é descartada, nunca um meio-termo furado. É essa propriedade que se chama **sampling consistente**.

## 4. Tail sampling: decide no final, vê o trace inteiro, custa caro em infra

Decisão tomada **depois** de ver "all or most of the spans within the trace" — só é possível decidir depois que (quase) tudo já aconteceu.

Isso desbloqueia decisões que head sampling não consegue:

- **"Always sampling traces that contain an error"** — a motivação original da seção 2.
- Baseado em latência total do trace (guarda os traces lentos, descarta os rápidos e sem graça).
- **"Sampling traces based on the presence or value of specific attributes"** — ex: sempre guardar traces de um serviço recém-deployado, pra observar de perto.
- Taxas diferentes por serviço (alto volume sampleia menos, baixo volume sampleia mais).

O preço: pra decidir "no final", alguém precisa **guardar todos os spans daquele trace em algum lugar até o trace terminar** — isso exige "stateful systems that can accept and store a large amount of data", tipicamente dezenas a centenas de nós de compute em sistemas de alto volume. E frequentemente essa peça acaba sendo "vendor-specific technology" — cada backend implementa o tail sampling do seu jeito.

## 5. Onde cada um roda

Head sampling pode rodar no **SDK**, já na origem — é stateless, não precisa ver nada além do Trace ID. Tail sampling **precisa** rodar num ponto que enxergue todos os spans de um mesmo trace juntos — normalmente um **Collector** dedicado (Tail Sampling Processor), configurado de um jeito que garanta que spans do mesmo Trace ID cheguem sempre na mesma instância de Collector (é aqui que entra o **Load Balancing Exporter** que já mencionei no Tema 8, seção 3 — ele existe justamente pra rotear por Trace ID de forma consistente antes do tail sampling decidir).

Combinação híbrida pra sistemas de volume muito alto: head sampling logo na origem, reduzindo volume bruto primeiro, e tail sampling depois — "in the interest of protecting the telemetry pipeline from being overloaded." Não é um substituindo o outro, é uma cascata de filtros.

## 6. Conexão com meu stack

No Collector que eu for operar, os dois processors relevantes são o **Probabilistic Sampling Processor** (head) e o **Tail Sampling Processor**. Pra tail sampling funcionar de verdade num deployment com múltiplas réplicas de Collector, a peça que fecha o circuito é o Load Balancing Exporter roteando por Trace ID — sem isso, cada réplica só vê um pedaço aleatório de cada trace e a decisão "guardei porque teve erro" fica quebrada, voltando ao mesmo problema da seção 3.

## 7. O que vira pergunta de entrevista

- "Qual a diferença fundamental entre head e tail sampling?" → *quando* a decisão é tomada — início (sem ver o trace todo) vs fim (vendo o trace completo).
- "Por que sampling probabilístico usa hash do Trace ID em vez de sortear aleatoriamente a cada span?" → pra garantir que todos os serviços tomem a **mesma** decisão pro mesmo trace — senão a árvore fica fragmentada, com buracos.
- "Como você garantiria 'sempre guardar traces com erro' numa stack de alto volume?" → tail sampling — só é possível decidir isso depois de ver o trace inteiro, o que head sampling não consegue por definição.

Ref: https://opentelemetry.io/docs/concepts/sampling/
