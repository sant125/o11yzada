# Tema 11 · Collector a fundo (receivers, processors, exporters, pipelines)

> `TL;DR:` O Collector não é uma caixa preta — é uma composição de peças pequenas e trocáveis: **receiver** (entra dado), **processor** (transforma, em sequência), **exporter** (sai dado), amarrados numa **pipeline** por sinal (traces/metrics/logs). **Connector** é a peça que junta duas pipelines, agindo como exporter de uma e receiver de outra ao mesmo tempo — é assim, por exemplo, que trace vira métrica sem reinstrumentar nada.

---

## 1. A ideia antes do nome: pipeline Unix, só que pra telemetria

Se você já usou `cat arquivo | grep erro | sort | uniq -c`, já entende a lógica do Collector: cada peça faz uma coisa, o dado flui de uma pra outra, e trocar uma peça não exige reescrever as outras. Receiver é o `cat` (traz o dado pra dentro), processor é o `grep`/`sort` (transforma no caminho), exporter é o destino final. A diferença é que aqui o "pipe" tem nome — **pipeline** — e é declarado em YAML, não encadeado na linha de comando.

> "A pipeline defines a path that data follows in the Collector: from reception, to processing (or modification), and finally to export."

## 2. Receiver: como o dado entra

> "Receivers typically listen on a network port and receive telemetry data. They can also actively obtain data, like scrapers."

Duas posturas possíveis: **push** (o receiver escuta uma porta e espera alguém mandar dado — ex: `otlp` receiver recebendo OTLP/gRPC na 4317) ou **pull/scrape** (o receiver vai buscar ativamente — ex: `prometheus` receiver fazendo scrape de um endpoint `/metrics`, do mesmo jeito que o Prometheus server já faz). O mesmo receiver pode alimentar **múltiplas pipelines** ao mesmo tempo — reaproveitando a mesma fonte de dado pra fluxos diferentes.

## 3. Processor: transforma, em sequência, na ordem que você escreve

> "A pipeline can contain sequentially connected processors."

**Sequencialmente** é a palavra que importa — a ordem que você lista os processors no YAML é a ordem que eles rodam, um alimentando a entrada do próximo. Alguns processors centrais:

- **`memory_limiter`** — protege o próprio Collector de estourar memória (a "auto-observabilidade" que apareceu no cap 08, seção 0 como valor de engenharia — o Collector cuidando de si mesmo antes de cuidar do seu dado).
- **`batch`** — o mesmo padrão do `BatchSpanProcessor` do Tema 9, só que no nível do Collector: agrupa antes de exportar, em vez de export imediato span a span.
- **`probabilisticsampler`** / **tail sampling processor** — as duas estratégias do Tema 6, implementadas como processors.
- **`attributes`** / **`filter`** / **`transform`** — adicionam, removem ou reescrevem atributos e dados no meio do caminho (isso é OTTL — Tema 12 do curso, transformação declarativa de telemetria; fica pra outra sessão, é um tópico com peso próprio).

## 4. Exporter: como o dado sai — e por que dá pra ter vários na mesma pipeline

> "Exporters typically forward the data they get to a destination on a network, but they can also send the data elsewhere."

O detalhe que fecha o pitch do Tema 8 (vendor-neutral, trocar backend sem reinstrumentar): **múltiplos exporters do mesmo tipo, ou de tipos diferentes, podem coexistir na mesma pipeline**, cada um mandando pra um destino distinto. Isso não é só teoria — é literalmente como você migra de backend sem downtime: liga um segundo exporter apontando pro novo destino, roda os dois em paralelo um tempo validando, depois desliga o antigo. Zero mudança na instrumentação, só configuração do Collector.

## 5. Pipeline: o YAML que amarra tudo, um por sinal

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp, zipkin]
      processors: [memory_limiter, batch]
      exporters: [otlp, zipkin]
```

Uma pipeline existe por tipo de sinal (traces, metrics, logs) — cada uma com sua própria lista de receivers/processors/exporters. Antes de eu te dar o detalhe que costuma pegar gente de surpresa: se o **mesmo** receiver alimenta **duas** pipelines diferentes (ex: um receiver `otlp` compartilhado entre a pipeline de traces e uma pipeline de debug/replicação), e um processor **numa** dessas pipelines trava (fica lento, bloqueado esperando algo) — o que você acha que acontece com a **outra** pipeline que compartilha aquele mesmo receiver?

### Resolução

> "The Collector creates only one receiver instance at runtime that sends the data to a fan-out consumer." [...] "if one processor blocks the call, the other pipelines attached to this receiver are blocked from receiving the same data."

Só existe **uma instância** do receiver rodando, e ela distribui (fan-out) pro consumidor de cada pipeline. Se um processor numa pipeline trava a chamada, as **outras** pipelines que compartilham aquele receiver ficam **bloqueadas** também — o fan-out não isola falha entre pipelines irmãs que compartilham a fonte. Implicação prática de arquitetura: um processor mal configurado (ou um exporter lento sem timeout, numa pipeline de debug que ninguém tava prestando atenção) pode derrubar a pipeline de produção que só está "vizinha" por compartilhar o receiver. Cuidado ao decidir o que compartilha receiver com o quê.

## 6. Connector: a peça que junta duas pipelines

> "Connectors join two pipelines, acting as both exporter and receiver. A connector consumes data as an exporter at the end of one pipeline and emits data as a receiver at the beginning of another pipeline."

O caso de uso mais valioso na prática: gerar **métrica a partir de trace**, sem reinstrumentar nada. Exemplo oficial, o connector `count`, contando span events que batem uma condição:

```yaml
connectors:
  count:
    spanevents:
      my.prod.event.count:
        description: The number of span events from my prod environment.
        conditions:
          - 'attributes["env"] == "prod"'
          - 'name == "prodevent"'
```

Aqui o connector age como **exporter** da pipeline de traces (consome os spans) e como **receiver** da pipeline de métricas (emite a contagem) — os dois papéis, ao mesmo tempo, na mesma peça. É assim, por exemplo, que dá pra ter métrica RED (rate/errors/duration) derivada direto do trace, sem instrumentar métrica separadamente pra cada serviço.

## 7. Padrões de deployment: Agent vs Gateway

- **Agent** — roda perto da aplicação (sidecar, daemonset por nó), recolhe da lib e já exporta pra fora ou pro Gateway.
- **Gateway** — recebe de vários Agents/bibliotecas, centraliza processamento pesado (tail sampling, por exemplo, que precisa ver o trace inteiro — Tema 6, seção 5) antes de exportar pro backend final.

## 8. Conexão com meu stack

Receiver `prometheus` (modo scrape/pull) é o caminho natural pra eu trazer o que já tenho hoje pra dentro do fluxo OTel sem reinstrumentar nada — aponta o Collector pros mesmos endpoints `/metrics` que o Prometheus já faz scrape. E o cuidado da seção 5 (fan-out bloqueando pipeline vizinha) é exatamente o tipo de coisa que só aparece em produção sob carga — vale desenhar a topologia de pipeline pensando nisso desde o início, não descobrir no incidente.

## 9. O que vira pergunta de entrevista

- "Qual a diferença entre processor e connector?" → processor transforma dado **dentro** de uma pipeline; connector **liga** duas pipelines, agindo como exporter de uma e receiver da outra ao mesmo tempo.
- "Como você geraria métrica a partir de dado de trace sem reinstrumentar a aplicação?" → um connector (ex: spanmetrics/count) na saída da pipeline de traces, alimentando a pipeline de métricas.
- "Por que a ordem dos processors no YAML importa?" → eles rodam sequencialmente, cada um recebendo a saída do anterior — trocar a ordem muda o resultado (ex: sampler antes ou depois de enriquecer atributo muda o que é avaliado).

Refs: https://opentelemetry.io/docs/collector/architecture/ · https://opentelemetry.io/docs/collector/configuration/#connectors
