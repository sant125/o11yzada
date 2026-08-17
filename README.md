# OpenTelemetry & Observability, destrinchado

Notas de estudo de observabilidade e [OpenTelemetry](https://opentelemetry.io/docs/), rumo ao [OTCA](https://training.linuxfoundation.org/certification/opentelemetry-certified-associate-otca/) e, mais importante, a ficar diferenciado em SRE/o11y. Não é resumo: é o assunto reexplicado até eu entender de verdade, com analogia antes de fórmula e jargão só depois da ideia.

Irmão do [srezada](../srezada) (SRE Workbook destrinchado). Lá é confiabilidade; aqui é enxergar o sistema por dentro a partir de fora.

Roteiro de referência (não é pra assistir aula por aula — só pra não deixar buraco de conteúdo core): [roteiro-curso-otca.md](roteiro-curso-otca.md), baseado no [prep course OTCA da KodeKloud](https://learn.kodekloud.com/learn/courses/prep-course-opentelemetry-certified-associate-certification-otca).

## Formato de cada tema

- A ideia central primeiro, o nome técnico depois
- Analogias e exemplos numéricos antes de definição formal
- Conexão com meu stack (Prometheus, Loki, Grafana, Kubernetes, Go, AWS/GCP)
- O que isso vira em entrevista

## Índice

**Parte I: Fundamentos**

- [x] [01 · Observabilidade: os sinais e por que pilar isolado não é observabilidade](01-observabilidade-sinais-e-correlacao.md)
- [x] [02 · Traces & Spans a fundo](02-traces-spans-a-fundo.md)
- [x] [03 · Metrics (tipos, cardinalidade, exemplars)](03-metrics-tipos-cardinalidade-exemplars.md)
- [x] [04 · Logs (estruturado, correlação, custo)](04-logs-estruturado-correlacao-custo.md)
- [x] [05 · Context propagation & Baggage](05-context-propagation-baggage.md)
- [x] [06 · Sampling (head vs tail)](06-sampling-head-vs-tail.md)
- [x] [07 · Semantic conventions](07-semantic-conventions.md)

**Parte II: OpenTelemetry**

- [ ] 08 · Arquitetura OTel (API, SDK, Collector)
- [x] [09 · Instrumentation (auto vs manual)](09-instrumentacao-auto-vs-manual.md)
- [x] [10 · OTLP (o protocolo)](10-otlp.md)
- [x] [11 · Collector a fundo (receivers, processors, exporters, pipelines)](11-collector-a-fundo.md)

**Parte III: Prática / stack**

- [ ] 12 · Backends de trace (Tempo, Jaeger) e correlação no Grafana
- [ ] 13 · Profiling contínuo (o quarto sinal)
- [ ] 14 · Montando a stack o11y num cluster Kubernetes
