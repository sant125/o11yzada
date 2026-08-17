# Tema 4 · Logs (estruturado, correlação, custo)

> `TL;DR:` Logs são o único sinal que o OTel **não** te faz reescrever do zero — em vez de trocar toda chamada `logger.info()` por uma API nova, o OTel entra como **bridge** por trás da lib de logging que você já usa, injetando `trace_id`/`span_id` automaticamente em todo log emitido dentro de um span ativo. O que muda de verdade é exigir schema **estruturado** e estável, não só "log em JSON" — e é volume de log, não trace, que geralmente domina o custo de uma stack de observabilidade.

---

## 1. A ideia antes do nome: logging já existia — OTel não reinventa, costura

Trace e métrica são sinais que você **ativamente chama** pra gerar (`tracer.start_span()`, `counter.add()`) — não existiam antes do OTel, então a API é o ponto de entrada natural. Logging é diferente: toda linguagem já tem décadas de libs de logging estabelecidas (`logging` do Python, `slog`/`log` do Go, Log4j/SLF4J do Java), e ninguém vai reescrever milhares de linhas de `logger.info(...)` espalhadas pra usar uma API nova. Por isso a arquitetura de log do OTel é deliberadamente diferente da de trace/métrica:

> "OpenTelemetry [is] designed to work with the logs you already produce, offering tools to correlate logs with other signals, add contextual attributes, and normalize different sources."

Na prática: você continua chamando `logger.info("pedido criado")` normalmente — quem muda é o **appender/bridge** por trás, que intercepta essa chamada e a transforma num LogRecord OTel, sem você reescrever nada.

## 2. Log Record: o modelo de dados

Um log é "a timestamped recording of an event". A distinção que importa: **"Not all logs are events, but all events are logs."** — Log é o conjunto amplo; Event (identificado pelo campo `EventName`) é um subconjunto com um tipo/classe nomeada e semântica mais estrita, tipicamente usado quando o dado importa mais pela sua **categoria** do que pela mensagem em si (ex: `EventName: "user.login"` com attributes estruturados, em vez de uma frase livre).

> Cuidado com colisão de nome: isso **não** é o mesmo "Event" do Tema 2 (span event). São dois conceitos de "evento" no OTel, em sinais diferentes — span event é uma anotação dentro de um span específico; log event é uma categoria de log record. Contexto resolve qual é qual, mas vale checar sempre qual sinal tá sendo discutido.

Campos do LogRecord:

- **Timestamp** — "time when the event occurred."
- **ObservedTimestamp** — "time when the event was observed" (pelo coletor/processo de ingestão).
- **TraceId / SpanId / TraceFlags** — correlação com trace (seção 3).
- **SeverityText / SeverityNumber** — nível (`INFO`, `ERROR`...) em texto e em número (número existe pra comparação/filtro consistente entre linguagens que nomeiam níveis diferente).
- **Body** — o conteúdo do log em si.
- **Resource** — de onde veio (mesmo conceito do Tema 7 — `service.name`, etc).
- **InstrumentationScope** — qual lib/componente emitiu.
- **Attributes** — metadado extra estruturado.
- **EventName** — ver parágrafo acima.

Antes de eu te dar o motivo: por que existem **dois** timestamps — `Timestamp` (quando aconteceu) e `ObservedTimestamp` (quando foi observado)? Pensa numa situação onde uma aplicação bufferiza logs em memória e só manda em lote de tempos em tempos, ou um sistema que só processa/ingere logs importados de um arquivo horas depois de gerados.

### Resolução

Se só existisse um timestamp, um pipeline de ingestão atrasado (log gerado agora, processado daqui 10 minutos por causa de buffer/fila) ia confundir "quando o evento realmente aconteceu" com "quando o sistema de observabilidade ficou sabendo dele" — isso quebra ordenação temporal correta numa investigação (você quer saber *quando de fato* aconteceu, não quando o pipeline te contou). Separar os dois deixa explícito: `Timestamp` é a verdade sobre o evento, `ObservedTimestamp` é sobre a latência do próprio pipeline de telemetria — útil, inclusive, pra diagnosticar atraso na ingestão de log em si.

## 3. Correlação automática com trace: o que fecha o círculo do Tema 5

> "OpenTelemetry will automatically correlate your existing logs with any active trace and span, wrapping the log body with their IDs."

Se um log é emitido **dentro** de um span ativo (ex: dentro do handler de uma request que já tem um trace em andamento), o `trace_id`/`span_id` daquele span são injetados automaticamente no LogRecord — sem você escrever esse encanamento manualmente. É o Tema 5 (context propagation) aparecendo de novo, agora não entre processos, mas entre **sinais dentro do mesmo processo**: o Context ativo (que carrega o Span Context) é o que o bridge de logging lê pra saber "esse log pertence a esse trace/span".

É exatamente esse par `trace_id`/`span_id` no log que faz o "clica no log e vai pro trace" (ou o inverso) funcionar no Grafana — Loki e Tempo correlacionando via esses campos, não por timestamp aproximado ou heurística de texto.

## 4. Estruturado, semiestruturado, não-estruturado — o schema é o que importa, não o formato

- **Não-estruturado** — "don't follow a consistent structure", texto livre. "Much more difficult to parse and analyze at scale" — inadequado pra observabilidade séria em produção.
- **Semiestruturado** — "include machine-readable key/value pairs" (ex: JSON), mas "do not guarantee a stable schema across emitters" — cada linha pode ter chaves diferentes, tipos diferentes pro mesmo campo. É JSON, mas não é confiável pra parsear em escala.
- **Estruturado** — tem "defined, consistent schema or typed fields that downstream systems can reliably parse and interpret." O que diferencia não é "é JSON", é **schema estável** — o campo `status_code` é sempre inteiro, sempre presente, sempre com esse nome, em todo log daquele tipo.

Erro comum: achar que "log em JSON" já é log estruturado. Sem schema consistente entre as linhas, é semiestruturado — parseia, mas não confia.

## 5. Arquitetura: Log Appender/Bridge vs uso direto da API

Dois caminhos, quem escreve cada um:

- **Log Appender/Bridge** — construído por **autores de biblioteca de logging** ("logging library authors build log appenders/bridges"), não por você. É o handler/adapter que pluga na lib de log que sua aplicação já usa (`logging.Handler` no Python, por exemplo) e converte cada chamada em LogRecord OTel automaticamente.
- **Logs API direta** — "The Logs API is public and can be used directly by application code or indirectly via existing logging libraries." Uso direto é raro — normalmente só faz sentido pra quem tá escrevendo uma lib de logging nova do zero, não pra aplicação final.

Na prática, quase sempre você configura o bridge uma vez (nível de framework/bootstrap da aplicação) e nunca mais toca em código de log — os `logger.info/warning/error` continuam exatamente como estavam.

## 6. Custo: log costuma ser o sinal mais caro em volume

Diferente de trace (amostrado — Tema 6) e métrica (agregada por natureza), log tende a ser o sinal que ninguém sampleia por padrão — toda linha que a aplicação emite vira um LogRecord. Em produção com volume alto, log não-filtrado costuma ser **o maior custo de armazenamento** da stack inteira, não trace. As alavancas práticas de controle: nível de severidade (não manda `DEBUG` pra produção sem motivo), atributo estruturado em vez de string concatenada (reduz parsing e indexação cara no backend), e — quando o volume justifica — sampling de log também, não só de trace.

## 7. Conexão com meu stack

Loki é meu backend de log hoje. A correlação da seção 3 (`trace_id`/`span_id` injetado automaticamente) é o que faz o botão "Related traces" do Loki no Grafana funcionar quando o Tempo entrar — hoje sem Tempo, esse campo já existe no log, só não tem pra onde apontar ainda (mesmo ponto que fechei no Tema 3 sobre exemplars). E o cuidado de custo da seção 6 é ainda mais direto pra mim: Loki indexa por label, então atributo de alta cardinalidade em log tem o mesmo problema do Tema 3 — só que em label de log, não em métrica.

## 8. O que vira pergunta de entrevista

- "Por que a arquitetura de log do OTel é diferente da de trace/métrica?" → logging já existia antes do OTel em toda linguagem; a estratégia é bridge por trás da lib existente, não uma API nova pra reescrever chamadas.
- "O que faz um log ser 'estruturado' de verdade, além de estar em JSON?" → schema **estável e consistente** entre emissores — tipo e presença do campo previsíveis, não só sintaxe JSON válida.
- "Como um log se correlaciona com um trace automaticamente?" → injeção automática de `trace_id`/`span_id` quando o log é emitido dentro de um span ativo, via o mesmo Context do Tema 5.

Ref: https://opentelemetry.io/docs/concepts/signals/logs/
