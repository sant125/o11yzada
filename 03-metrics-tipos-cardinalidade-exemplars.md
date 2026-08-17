# Tema 3 · Metrics (tipos, cardinalidade, exemplars)

> `TL;DR:` Métrica é o sinal barato e comprimido — número agregado ao longo do tempo, não o detalhe de cada request individual (isso é trace). Tem 4 instrumentos-base (Counter, UpDownCounter, Gauge, Histogram, cada um com variante síncrona/assíncrona), cada medição carrega atributos que formam a **cardinalidade** da métrica — e cardinalidade alta é o jeito mais comum de estourar memória de um sistema de métricas. **Exemplar** é o que costura métrica de volta em trace específico, sem isso métrica e trace vivem em universos separados.

---

## 1. A ideia antes do nome: por que métrica é barata e trace é caro

Retomando o Tema 1, seção 3: cada sinal tem um custo. Trace guarda o detalhe de **cada** request — caro, mas você pode perguntar "o que aconteceu *nesse* request específico". Métrica já chega **agregada**: você perde o detalhe individual, mas ganha barato o suficiente pra manter em alta resolução por muito tempo, e pra fazer alerta em cima (`error_rate > 5%` por 5 minutos, por exemplo). Trace responde "o que aconteceu aqui". Métrica responde "quanto, com que frequência, isso tá subindo ou descendo" — sem custar uma fortuna de armazenamento.

> "A metric is a measurement of a service captured at runtime. [...] a metric event consists of the measurement itself, the time it was captured, and associated metadata."

## 2. Os 4 instrumentos — e a diferença de "o que cada um promete"

Cada instrumento não é só "um jeito de guardar número" — é uma **promessa de comportamento** que quem consome a métrica depois pode confiar:

- **Counter** — "A value that accumulates over time -- you can think of this like an odometer on a car; it only ever goes up." Exemplo: total de requests processadas. **Monotônico**: nunca diminui.
- **UpDownCounter** — "A value that accumulates over time, but can also go down again. An example could be a queue length." Exemplo: tamanho de fila, conexões ativas. **Não-monotônico**: sobe e desce.
- **Gauge** — "Measures a current value at the time it is read. An example would be the fuel gauge in a vehicle." Não acumula nada — é a leitura instantânea de agora. Exemplo: uso de memória neste segundo.
- **Histogram** — "A client-side aggregation of values, such as request latencies. A histogram is a good choice if you are interested in value statistics." Não é um número, é uma distribuição — quantas requests caíram em cada faixa de latência.

Cada um tem uma variante **assíncrona** (Asynchronous Counter/UpDownCounter/Gauge) — a diferença não é o que medem, é **quando**: a versão síncrona registra a cada evento que acontece no código (cada request incrementa o Counter ali mesmo); a versão assíncrona só é "coletada uma vez por ciclo de export" — você registra um callback que lê o valor atual (ex: tamanho de uma fila lida de uma lib externa) e o SDK só chama esse callback na hora de exportar, não a cada mudança.

Por que a distinção monotônico/não-monotônico importa de verdade: um consumidor de métrica (Prometheus, por exemplo) trata Counter e UpDownCounter de formas matematicamente diferentes — `rate()` só faz sentido matemático em cima de algo que só sobe; aplicar `rate()` num UpDownCounter que oscila pra baixo dá resultado sem significado.

## 3. Temporality: delta vs cumulative — o detalhe que quebra silenciosamente com Prometheus

Pra instrumentos síncronos, existe uma escolha de **temporality** que decide como o estado é mantido entre ciclos de export:

- **Cumulative** — "the SDK retains state across cycles" — o valor exportado é sempre "o total desde o início", cada export é maior (ou igual) que o anterior.
- **Delta** — "the SDK resets state after each cycle" — o valor exportado é só "o que aconteceu desde o último export", reseta a cada ciclo.

Aqui vale pensar antes de eu resolver: o Prometheus, historicamente, é construído em cima de contadores **cumulativos** — o cliente nunca reseta, e é o **servidor** Prometheus que calcula taxa de variação via `rate()` na hora da query, detectando reset de contador (deploy reiniciou o processo, contador voltou a zero) como caso especial. Se um SDK OTel exporta um Counter em temporality **delta** direto pra um endpoint que espera semântica cumulativa (ex: Prometheus remote-write), o que você acha que acontece com o número que chega lá — ele ainda representa "total acumulado", ou virou outra coisa?

### Resolução

Delta chegando onde cumulative é esperado vira **"o incremento daquele intervalo apenas"** — não é mais um total crescente, é um valor que sobe e desce a cada ciclo dependendo de quanto aconteceu naquela janela. Se um sistema espera cumulative e aplica a mesma lógica de `rate()`/reset-detection em cima de um valor que na verdade já é delta, a matemática sai errada — sem erro, sem alerta, só um gráfico que não bate com a realidade. É por isso que o **Collector** (Tema 8) frequentemente entra na conversa aqui também: o exporter Prometheus do Collector converte temporality delta→cumulative antes de entregar, exatamente pra essa incompatibilidade não vazar pro seu Grafana. Moral prática: ao configurar exporter de métrica, checar **qual temporality ele produz** e **o que o backend de destino espera** — não é assumido, é configuração explícita.

## 4. Atributos, cardinalidade, e o teto de proteção

**Cardinalidade** é o número de combinações únicas de atributos que uma métrica acumula:

> "The cardinality of a metric is the number of unique attribute combinations reported for it."

Cada atributo que você anexa a uma medição (`http.route`, `user.tier`, `status_code`) multiplica combinações possíveis. O erro clássico: anexar um atributo de **alta cardinalidade** — `user.id` bruto, URL completa não normalizada — faz o número de combinações únicas crescer sem limite (um valor novo por usuário, por request), e o SDK mantém agregação **separada pra cada combinação**. Isso não é hipotético: é a causa mais comum de custo/memória explodindo num backend de métrica.

O SDK do OTel tem uma defesa embutida — **cardinality limit**, default de **2000** combinações por stream de métrica, configurável via Views:

> "their measurements are aggregated into a single overflow data point identified by the attribute `otel.metric.overflow=true`."

Propriedades importantes desse comportamento:
- **Nenhuma medição é perdida** — só os atributos são descartados; o valor recolhido continua contando, só que jogado no "balde de overflow".
- **Memória é limitada** — o SDK nunca rastreia mais que o limite configurado.
- **Overflow é observável** — todo SDK usa o mesmo marcador `otel.metric.overflow=true`, então dá pra detectar que isso aconteceu.

Detalhe que vale saber: esse limite **não** se aplica a atributos de Resource (`service.name`) nem ao escopo de instrumentação — só aos atributos da medição em si.

## 5. Exemplars: a costura entre métrica e trace

> "An exemplar is a recorded value that associates OpenTelemetry context to a metric event within a Metric."

Um exemplar é anexado a um data point de métrica (tipicamente um bucket de histograma) e carrega:

- `trace_id` / `span_id` (opcional) — o trace específico que gerou aquela medição.
- `time_unix_nano` — quando.
- `value` — o valor observado.
- `filtered_attributes` — atributos que não entraram na agregação principal, mas dão contexto extra.

> "One use case is to allow users to link Trace signals w/ Metrics."

Na prática: você olha um histograma de latência no Grafana, vê um bucket alto em "P99 > 2s", e em vez de só saber "teve requests lentas", o exemplar te dá o `trace_id` de uma request **específica** que caiu naquele bucket — clica e vai direto pro trace no Tempo. É a métrica (barata, agregada) apontando de volta pro trace (caro, detalhado) exatamente no ponto que interessa, sem precisar guardar trace de tudo.

## 6. Conexão com meu stack

Já rodo Prometheus — a pegadinha da seção 3 (delta vs cumulative) é risco real, não teórico, porque é literalmente o modelo que meu backend de métrica assume. Quando eu configurar exporter de métrica OTel apontando pro meu Collector → Prometheus, checar explicitamente a temporality configurada é passo obrigatório, não opcional. E exemplars é o motivo concreto de eu querer o Tempo (ainda "planejado" hoje) rodando: sem backend de trace, o `trace_id` no exemplar não tem pra onde apontar.

## 7. O que vira pergunta de entrevista

- "Diferença entre Counter e UpDownCounter?" → monotonicidade — Counter só sobe (`rate()` faz sentido), UpDownCounter sobe e desce (tamanho de fila, por exemplo).
- "O que é cardinalidade alta e por que é perigosa?" → cada combinação única de atributo vira uma série rastreada separadamente; atributo tipo `user.id` bruto faz isso crescer sem limite e estourar memória.
- "Como o OTel SDK se protege de cardinalidade descontrolada?" → cardinality limit (default 2000 por stream), overflow marcado com `otel.metric.overflow=true`, sem perder o valor — só os atributos daquela combinação em excesso.
- "Pra que serve um exemplar?" → linka um data point de métrica a um trace específico (`trace_id`/`span_id`), permitindo ir do agregado pro caso concreto.

Refs: https://opentelemetry.io/docs/concepts/signals/metrics/ · https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars
