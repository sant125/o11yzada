# Tema 2 · Traces & Spans a fundo

> `TL;DR:` Trace é a árvore que representa o caminho de uma request; span é cada nó dessa árvore — uma unidade de trabalho com início, fim, e um `SpanContext` (trace id + span id) que amarra tudo. Span tem 5 "kinds" que descrevem o *papel* da operação (SERVER, CLIENT, INTERNAL, PRODUCER, CONSUMER), attributes pra metadado estático, events pra "aconteceu algo neste instante", links pra relação causal fora da árvore de pai/filho, e status pra dizer se deu certo.

---

## 1. A ideia antes do nome: trace é árvore, span é nó

Um **trace** não é uma linha do tempo plana — é uma **árvore**. A raiz é a primeira operação (ex: request chegando no gateway), e cada chamada que ela dispara (pro serviço B, pro banco, pra fila) vira um filho. Cada nó dessa árvore é um **span**: uma unidade de trabalho com hora de início, hora de fim, e um vínculo explícito com quem o chamou (o `parent`).

> "A span represents a unit of work or operation."

O que amarra um span a um trace e a seu pai é o **Span Context** — um objeto imutável presente em todo span:

> "Span Context [is] an immutable object on every span [containing] Trace ID, Span ID, Trace Flags, and Trace State."

- **Trace ID** — mesmo em todos os spans da mesma árvore (é o que permite reconstruir a árvore inteira depois).
- **Span ID** — único daquele span específico.
- **Trace Flags** — flags binárias (ex: se esse trace foi sampleado).
- **Trace State** — pares chave-valor específicos de vendor, viaja junto mas é opcional/extensível.

O Span Context é o que viaja de processo pra processo via propagação de contexto (Tema 5) — é literalmente o "bilhete" que cada serviço recebe e usa pra saber "eu sou filho de tal span, desse trace".

## 2. Os campos que compõem um span

Além do Span Context e do vínculo com o pai (vazio só na raiz), um span carrega:

- **Name** — identificador da operação (ex: `GET /checkout`, `SELECT orders`).
- **Timestamps** — `start_time` e `end_time`. É daqui que vem a latência — `end_time - start_time` de cada span, e a soma/superposição deles no trace inteiro é o que o waterfall do Grafana Tempo/Jaeger desenha.
- **Attributes** — metadado key-value (seção 4).
- **Events** — anotações com timestamp próprio dentro do span (seção 4).
- **Links** — relação causal com spans fora da árvore de pai/filho (seção 5).
- **Status** — sucesso/erro (seção 6).
- **Kind** — o papel da operação (seção 3).

## 3. Span Kind: o papel da operação, não o "tipo de serviço"

Kind não descreve a tecnologia — descreve **o papel daquela chamada específica** na comunicação entre processos. Os cinco valores:

- **SERVER** — "a synchronous incoming remote call such as an incoming HTTP request." A ponta que **recebe**.
- **CLIENT** — "a synchronous outgoing remote call such as an outgoing HTTP request or database call." A ponta que **chama**.
- **INTERNAL** — "operations which do not cross a process boundary", tipo chamada de função dentro do mesmo processo. Não cruza rede.
- **PRODUCER** — "the creation of a job which may be asynchronously processed later." Publicar numa fila, por exemplo.
- **CONSUMER** — "the processing of a job created by a producer", que pode rodar muito depois, em processo totalmente diferente.

Por que isso importa na prática: um request HTTP síncrono gera um par **CLIENT** (de quem chamou) + **SERVER** (de quem recebeu) com o mesmo trace id, formando parent/child direto. Já uma mensagem de fila gera um **PRODUCER** (publicou) e, minutos depois, um **CONSUMER** (processou) — sem relação síncrona de pai/filho no tempo, mas ainda no mesmo trace, ligados por **link** (seção 5), não por parentesco direto.

## 4. Attributes vs Events: metadado estático vs "aconteceu agora"

**Attributes** são pares chave-valor de metadado sobre a operação inteira:

> "Key-value pairs that contain metadata... keys must be non-null strings, values: string, boolean, floating point, integer, or arrays of these."

Exemplo: `http.response.status_code=500`, `db.system=postgresql` — descreve a operação como um todo, não um instante dela.

**Events** são diferentes — têm timestamp próprio, marcam um momento específico dentro da duração do span:

> "A Span Event can be thought of as a structured log message (or annotation) on a Span, [marking] a meaningful, singular point in time."

Regra prática pra decidir qual usar: **se o timestamp em si é informação relevante, é event; se não é, é attribute.** Exemplo: "cache miss aconteceu aos 340ms dentro desse span de 500ms" → event (o *quando* dentro do span importa). "Esse span rodou contra o banco `orders-db`" → attribute (não importa em que milissegundo isso foi verdade, é verdade o span inteiro).

## 5. Span Links: relação causal sem ser pai/filho

Links "associate one span with one or more spans, implying a causal relationship" — mas fora da árvore normal de parent/child. O caso clássico é exatamente o PRODUCER/CONSUMER da seção 3: o consumer que processa uma mensagem de fila não é "filho cronológico" do producer (pode rodar segundos, minutos, até horas depois, em processo completamente desacoplado) — mas ainda é **causalmente** relacionado a ele. Link registra esse "isso aconteceu por causa daquilo", sem forçar o modelo de árvore síncrona.

Outro caso comum: batch processing, onde um span processa itens que vieram de N traces diferentes — ele linka pra cada trace de origem, porque não é filho de nenhum sozinho, é resultado de vários.

## 6. Span Status: só três valores, e o padrão já é "deu certo"

- **Unset** — o padrão. Span terminou sem erro relatado.
- **Ok** — marcado explicitamente como sucesso pelo desenvolvedor.
- **Error** — erro ocorreu.

Detalhe que economiza código: "it is not necessary to explicitly mark a span as Ok" — no caso comum, você só mexe no status quando dá **erro**; deixar `Unset` já comunica sucesso. Setar `Ok` manualmente é raro, geralmente reservado pra quando você quer sobrescrever uma inferência de erro que a instrumentação automática fez errado.

A relação com exceptions: uma exceção não vira status sozinha — o padrão é capturar a exceção como um **event** (com stack trace nos attributes do event) **e** setar status `Error` no span, os dois juntos. Isso é o que dá pro Tempo/Jaeger mostrar "esse span dourado no waterfall tem um X vermelho — clica e vê o stack trace".

## 7. Conexão com meu stack

No Grafana Tempo, o waterfall que eu abro numa trace é literalmente essa árvore de spans desenhada: cada barra é um span, a posição/tamanho vem de `start_time`/`end_time`, a cor de erro vem do status, e o texto que aparece ao passar o mouse é a mistura de name + attributes + events daquele span. Entender attributes vs events na prática muda como eu leio um trace: attribute é o que filtra/agrupa na busca (`db.system=postgresql`), event é o que eu leio dentro de um span específico pra achar o "aconteceu isso nesse ponto exato".

## 8. O que vira pergunta de entrevista

- "Qual a diferença entre span attribute e span event?" → attribute é metadado da operação inteira, event é algo pontual com timestamp próprio dentro do span.
- "Quando você usaria um span link em vez de parent/child normal?" → quando a relação é causal mas assíncrona/desacoplada — produtor/consumidor de fila é o exemplo canônico.
- "Por que a maioria dos spans nunca seta status Ok explicitamente?" → porque `Unset` já é o default de sucesso; você só intervém pra marcar `Error`.

Ref: https://opentelemetry.io/docs/concepts/signals/traces/
