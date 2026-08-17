# Tema 10 · OTLP (o protocolo)

> `TL;DR:` OTLP é o formato/transporte único que o OTel define pra telemetria trafegar entre quem gera (SDK), quem processa (Collector) e quem recebe (backend) — dois transportes intercambiáveis (gRPC porta 4317, HTTP porta 4318), sempre request/response, com regra clara de retry (só reenvia erro retentável, com backoff) e compressão opcional via gzip.

---

## 1. A ideia antes do nome: um protocolo só, pra não reescrever exporter pra cada par de ferramentas

Sem um protocolo comum, cada combinação de "quem gera → quem recebe" pede um formato próprio: um exporter pra falar com Jaeger, outro pra falar com Prometheus remote-write, outro pra falar com o agente proprietário de um vendor. Isso é o mesmo problema de vocabulário do Tema 07 (semantic conventions), só que uma camada abaixo — não é "como nomear o atributo", é "em que formato de bytes o atributo viaja pela rede".

Definição oficial:

> "OTLP specifies the encoding, transport, and delivery mechanism of telemetry data between sources, intermediate nodes and backends."

OTLP é justamente esse "protocolo único" — qualquer SDK, qualquer Collector, qualquer backend que fale OTLP entende o outro lado sem tradução. É o que permite a troca de exporter no Collector (Tema 8, seção 1) ser literalmente uma linha de config: os dois lados já concordam no formato.

## 2. Dois transportes, mesma semântica

OTLP não é um transporte único — são dois, escolhidos por conveniência de infra, não por diferença de capacidade:

- **OTLP/gRPC** — porta padrão **4317**. Mensagens `Export*ServiceRequest` (`ExportTraceServiceRequest`, `ExportMetricsServiceRequest`, `ExportLogsServiceRequest`), request/response unário, tamanho recomendado até **64 MiB** por requisição.
- **OTLP/HTTP** — porta padrão **4318**. `POST` com corpo Protobuf, em dois encodings: **binary** (`Content-Type: application/x-protobuf`) ou **JSON** (`Content-Type: application/json`, onde `traceId`/`spanId` viram string hex case-insensitive).

Na prática: gRPC costuma ganhar em ambiente que já é tudo gRPC internamente (menos overhead, multiplexing); HTTP ganha quando tem proxy/load balancer/firewall no meio que só entende HTTP puro (mais fácil de debugar com `curl`, mais fácil de passar por infra que não fala gRPC nativamente). A regra em qualquer um dos dois é a mesma: existe **um** tipo de mensagem, `Export`, e a resposta indica sucesso completo, sucesso parcial, ou falha.

## 3. Retry: nem todo erro merece reenvio

Antes de eu te dar a regra pronta — pensa comigo: se o Collector manda um lote de spans pro backend e recebe um erro, faz sentido reenviar **sempre**? E se o erro for "esse span tem um campo obrigatório faltando, dado malformado"? Reenviar o mesmo lote malformado dez vezes resolve alguma coisa, ou só desperdiça banda e atraso?

### Resolução: retentável vs não-retentável

A spec separa os dois casos:

- **Erro retentável** — o servidor tá temporariamente indisponível (sobrecarregado, reiniciando). Aqui sim vale reenviar, com **backoff exponencial** — espera crescente entre tentativas, pra não bater no servidor já sofrendo com mais carga ainda.
- **Erro não-retentável** — o dado em si é inválido. Reenviar o mesmo payload malformado não conserta nada; a spec não recomenda retry aqui.

Sinalização de backpressure difere por transporte: em **gRPC**, o servidor devolve código `Unavailable` com um `RetryInfo` (diz quanto tempo esperar). Em **HTTP**, é `429` (rate limit) ou `503` (indisponível), com header opcional `Retry-After`.

Isso conecta direto com o Tema 8, seção 1 (por que ter Collector): é exatamente essa lógica de retry/backoff que o Collector absorve **fora** do hot path do seu serviço — o SDK da aplicação não fica preso reenviando contra um backend fora do ar, quem lida com isso é o Collector.

## 4. Compressão

Requisito da spec: todo componente servidor **deve** suportar no mínimo duas opções — `none` (sem compressão) e `gzip`. Tanto gRPC quanto HTTP indicam isso via header/metadata apropriado. Na prática, gzip é o default sensato pra reduzir custo de banda em volume alto de telemetria (trace de sistema com tráfego pesado gera *muito* byte).

## 5. Conexão com meu stack

Prometheus/Loki/Tempo (via Grafana Cloud/Alloy ou Collector próprio) aceitam OTLP nativamente hoje — então o exporter do meu Collector fala OTLP/gRPC (4317) direto pro backend, sem tradução. Se um dia eu precisar debugar "por que a telemetria não tá chegando", a primeira pergunta prática é: 4317 (gRPC) ou 4318 (HTTP) tá aberto entre o Collector e o destino, e o erro que aparece no log é retentável (backend sobrecarregado) ou não (payload rejeitado)?

## 6. O que vira pergunta de entrevista

- "Por que dois transportes pro mesmo protocolo?" → mesma semântica, escolha por conveniência de infra — gRPC quando o ambiente já é gRPC-nativo, HTTP quando tem proxy/firewall no meio que só fala HTTP.
- "Como o Collector decide se reenvia um lote que falhou?" → depende se o erro é retentável (backend indisponível, backoff exponencial) ou não (dado inválido, reenviar não resolve).
- "Que portas são padrão pra OTLP?" → 4317 (gRPC), 4318 (HTTP).

Ref: https://opentelemetry.io/docs/specs/otlp/
