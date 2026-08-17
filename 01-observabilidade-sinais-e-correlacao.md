# Tema 1 · Observabilidade: os sinais e por que pilar isolado não é observabilidade

> `TL;DR:` Observabilidade é conseguir fazer qualquer pergunta sobre o estado interno do sistema olhando só as saídas dele, sem subir código novo pra responder. Os sinais de telemetria são traces, metrics e logs (profiling é o quarto entrando). Baggage NÃO é sinal — é context propagation, um meio pra enriquecer os outros. E o pulo do gato: ter os três sinais separados não é observabilidade, é três monitorings; o que vira observabilidade é a correlação entre eles.

---

## 1. A ideia antes do nome: observar é interrogar, não vigiar

Monitorar é ficar de olho no que você **já sabia** que podia dar errado: você previu que o disco pode encher, criou um alerta de disco, e o painel te avisa. Ótimo — pra falhas conhecidas. O problema é que produção não te avisa só das falhas que você previu. Ela te surpreende com a combinação que ninguém imaginou: aquele endpoint específico, só pra clientes de um plano, só quando o cache expira junto com um deploy. Pra esse tipo de coisa, painel pré-montado não serve — porque ninguém montou um painel pra uma pergunta que ninguém tinha feito ainda.

Observabilidade é a capacidade de responder essas perguntas novas **sem tocar no código**. A definição oficial do OpenTelemetry:

> "Observability lets you understand a system from the outside by letting you ask questions about that system without knowing its inner workings."

Entender o sistema **de fora**, fazendo perguntas, sem precisar conhecer as tripas dele. É exatamente o que você tinha intuído: sinais vitais pra entender o sistema sem ler a lógica linha por linha.

O par de conceitos que separa os dois:

- **Monitoring** = *known-unknowns*. Você sabe que aquilo pode falhar, só não sabe quando. Alerta e dashboard resolvem.
- **Observability** = *unknown-unknowns*. O problema inédito, que você nem sabia que era possível. Só resolve se as saídas do sistema forem ricas o bastante pra você interrogar depois do fato.

Ver também [srezada Cap 4 (Monitoring)](../srezada/04-monitoring.md) — lá é a mesma distinção, olhada do lado prático de SRE: os 5 usos de monitoring e por que só o alerta é a fatia *known-unknowns* dos 5.

Na prática: **monitoramento** é métrica predefinida, painel predefinido, problema conhecido — saturação de disco/rede/CPU/memória/inodes, `max_connections` estourando. Tudo que a gente já sabe que pode dar ruim, com threshold e alerta prontos. **Observabilidade** agrega esses sinais a outros sintomas: trace pra acompanhar todo o caminho da request ao longo do sistema, log pra correlação — e daí você extrai *onde*, *quando* e *como* ocorreu, não só o sinal cruzando um threshold predefinido. Isso te tira do puramente reativo; e mesmo quando você reage, tem uma gama muito maior de sinais pra destrinchar o problema.

Por que isso pesa mais em arquitetura distribuída: num monolito, o bug mora num processo, você anexa um debugger e pronto. Em microserviço, o bug quase nunca mora *dentro* de um serviço — mora no **caminho entre eles** (a chamada que ficou lenta, o retry que cascateou, o serviço C que só falha quando A e B batem juntos). Não dá pra anexar debugger no caminho entre 12 pods. Você só enxerga esse caminho se o sistema o **emitiu como telemetria**. Daí a necessidade dos sinais.

Ref: https://opentelemetry.io/docs/concepts/observability-primer/

## 2. Os sinais — e a pegadinha do baggage

Sinal (signal) é uma categoria de telemetria que o sistema emite. Os três clássicos:

- **Métrica** — número agregado ao longo do tempo (taxa de erro, uso de CPU, throughput).
- **Log** — mensagem com timestamp de um serviço, não necessariamente ligada a uma request específica.
- **Trace** — o caminho de uma request atravessando os serviços (detalhado na seção 3).

E **profiling** é o quarto sinal entrando na dança — perfilamento contínuo de CPU/memória por linha de código (fica pro Tema 13).

Agora a pegadinha importante. Muita gente lista **baggage** como se fosse um quarto pilar, do mesmo nível de trace/log/métrica. **Está errado.** Baggage não é uma saída que você observa — é um **mecanismo de transporte de contexto**. Definição do OTel:

> "Baggage is contextual information that is passed between signals. (...) it is a separate key-value store and is unassociated with attributes on spans, metrics, or logs without explicitly adding them."

Ou seja: baggage é um dicionário chave-valor que **viaja junto da request** atravessando os serviços. Exemplo concreto: o API gateway lá na borda sabe que `user.tier=premium`. O serviço de pagamento, cinco saltos adiante, não sabe disso — a request chegou lá pelada. Se o gateway põe `user.tier=premium` no baggage, esse par viaja com a request e o serviço lá no fundo consegue ler. Aí você adiciona esse valor como atributo no span/log dele, e agora dá pra filtrar "me mostra a latência só dos premium" no serviço inteiro.

O detalhe que quase todo mundo erra: **baggage não vira atributo sozinho.**

> "To add baggage entries to attributes, you need to explicitly read the data from baggage and add it as attributes to your spans, metrics, or logs."

Você tem que **explicitamente** ler do baggage e escrever como atributo. Baggage é o meio (carregar o dado através da malha); o atributo no sinal é o fim (o dado gravado, observável). Por isso baggage não é pilar: ele não é o que você observa, é o encanamento que leva contexto até onde você observa.

Cuidado de segurança que também cai (e é bom senso de SRE): baggage anda em **HTTP headers**, em texto, e não tem verificação de integridade.

> "Baggage and other parts of trace context are sent in HTTP headers, making it visible to anyone inspecting your network traffic. (...) there are no built-in integrity checks to ensure that Baggage items are yours."

Regra prática: **nunca** ponha PII, token ou segredo no baggage. Vaza no header e qualquer um no meio do caminho lê.

Refs: https://opentelemetry.io/docs/concepts/signals/ · https://opentelemetry.io/docs/concepts/signals/baggage/

## 3. Pra que cada sinal serve — e o que cada um custa

Cada sinal responde uma pergunta diferente e tem um custo diferente. Escolher o errado é lento e caro.

| Sinal | O que é | Pergunta que responde | Custo / limite |
|---|---|---|---|
| Métrica | número agregado no tempo | "TEM problema? em que escala?" | barata; presa por **cardinalidade** |
| Log | evento discreto com timestamp | "o que exatamente aconteceu ali?" | caro em **volume** e indexação |
| Trace | caminho de 1 request entre serviços | "ONDE no caminho travou?" | caro; por isso quase sempre **amostrado** |

**Métrica** é barata porque é agregada: em vez de guardar cada request, guarda um contador que só cresce e você deriva a taxa. O limite dela é **cardinalidade** — cada combinação nova de labels vira uma série temporal separada na memória do Prometheus. Uma métrica `http_requests_total{status, method}` com 5 status e 4 métodos = 20 séries, tranquilo. Agora bota um label `user_id` com 1 milhão de usuários e viram milhões de séries: o Prometheus derrete. Regra de ouro: label de métrica só pra coisa de baixa cardinalidade (status, método, rota — não id, não email, não trace_id).

**Log** é o evento cru, discreto. Responde "o que exatamente", mas é caro porque é volumoso e alguém precisa indexar pra buscar. Por isso Loki é esperto: ele **não** indexa o corpo do log inteiro, só um punhado de labels, e o resto fica comprimido em objeto barato. Você paga a busca no query time, não no ingest.

**Trace** responde a pergunta que nem métrica nem log respondem sozinhos: *em qual dos 12 serviços, e em qual trecho dentro dele, a request gastou o tempo?* Ele quebra a request em **spans** (próximo tema), e é caro justamente porque uma request pode gerar dezenas de spans — daí o sampling (seção 5).

O encaixe com teu próprio raciocínio de latência intermitente, que fecha a seção:

1. **Trace** primeiro — localiza o span que estourou. "A latência está no serviço de pagamento, no span da query ao banco."
2. **Métrica** — mostra se é padrão ou soluço. "Essa query está lenta pra 3% das requests desde as 14h, ou foi um pico isolado?"
3. **Log** — o porquê naquele ponto. "Naquele span, o log diz `connection pool exhausted`."

Trace diz *onde*, métrica diz *quanto/em que escala*, log diz *por quê*. Usar log pra achar *onde* (grep em 12 serviços) é o caminho lento que você quer aposentar.

Ref: https://opentelemetry.io/docs/concepts/signals/traces/

## 4. Pilar isolado não é observabilidade — correlação é o fim

Aqui está a tese do tema inteiro. Você pode ter Prometheus, Loki e um backend de trace, os três lindos, cada um no seu dashboard — e **ainda não ter observabilidade**. Se pra investigar um incidente você abre a métrica numa aba, o log noutra, o trace noutra, e fica **alt-tabando tentando adivinhar** qual log casa com qual pico casa com qual trace, você não tem observabilidade: você tem **três monitorings** morando em silos.

O que transforma três sinais em observabilidade é a **correlação** — os três amarrados pelo mesmo fio:

- **`trace_id` no log e no span.** O log daquela request carrega o mesmo `trace_id` do trace. Aí, do span ruim, você pula direto pro log exato daquela execução — sem grep, sem adivinhar.
- **Exemplars ligando métrica → trace.** Um exemplar é um ponteiro pendurado num ponto da métrica que diz "esse pico de latência aqui? foi *este* trace_id". Você clica no ponto alto do gráfico e cai no trace que o causou.
- **Semantic conventions.** Nomes de atributo padronizados (`http.request.method`, `service.name`, ...) pra que "método HTTP" se chame igual em todo serviço e time. Sem isso, um serviço grava `method`, outro `http_method`, outro `verb`, e a correlação quebra porque nada casa.

O fluxo de debug que a correlação te dá, sem sair do Grafana:

1. Alerta de burn rate dispara (isso vem do srezada — SLO gastando rápido).
2. Você abre a métrica de latência, vê o pico, **clica no exemplar** → cai no trace daquela request lenta.
3. No trace, vê o span vermelho: serviço de pagamento, query ao banco.
4. Do span, pelo `trace_id`, pula pro **log exato** daquela execução → `connection pool exhausted`.

Quatro passos, um fio só, zero adivinhação. *Isso* é observabilidade. Três abas desconexas não é.

Refs: https://opentelemetry.io/docs/concepts/semantic-conventions/ · https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exemplars

## 5. Por que trace tem sampling (o suficiente pra entrevista; Tema 6 aprofunda)

Guardar 100% dos traces em prod com tráfego alto é caro demais — rede, armazenamento e o backend não aguentam uma request virando dezenas de spans, milhões de vezes por hora. Então você **amostra**: guarda uma fração. A pergunta é *quando* decidir o que guardar.

- **Head-based sampling** — decide **no início**, na origem da request, tipo "guarda 10% aleatório". Definição OTel: *"a sampling decision as early as possible"*. Barato, simples, e dá pra fazer em qualquer ponto do pipeline. O problema: no início você **ainda não sabe** se a request vai dar erro ou ficar lenta. Então você pode jogar fora justamente o trace raro que deu ruim — que é o único que você queria ver.
- **Tail-based sampling** — decide **no fim**, depois que o trace inteiro terminou e você já viu todos os spans: *"the decision to sample a trace takes place by considering all or most of the spans within the trace."* Aí dá pra ser esperto: "guarda 100% dos traces com erro, 100% dos lentos, e só 1% dos normais". Pega o que importa. O custo: precisa **segurar (buffer) todos os spans** até o trace fechar pra então decidir — infra stateful, pesada, e cara.

O trade-off numa frase, que é o que a entrevista quer ouvir: **head é barato mas cego pro raro; tail vê o raro mas é caro e stateful.** Na prática se combina os dois (head pra proteger o pipeline de volume, tail pra garantir os erros).

Ref: https://opentelemetry.io/docs/concepts/sampling/

## 6. Conexão com meu stack

Estado atual nos meus clusters: eu já subi **Prometheus** (métrica) e **Loki** (log). Dois dos três sinais de pé. O buraco é **trace** — ninguém tem, e é justamente a peça que responde "onde no caminho".

Onde entra: um backend de trace. As duas opções de mercado são **Jaeger** e **Grafana Tempo**. Pra quem já tem Grafana + Loki como eu, **Tempo** encaixa natural: mesma casa, integra pra correlação nativa (do trace pro log via `trace_id`, do exemplar da métrica pro trace), e o storage dele é objeto barato (S3/GCS), sem precisar de Elasticsearch gordo como o Jaeger clássico pedia.

O pulo do gato, e é literalmente meu diferencial competitivo: **Prometheus + Loki + Tempo correlacionados no Grafana** — os três amarrados por `trace_id` e exemplars — é a stack de observabilidade *completa*, open source, que a **maioria das empresas não monta** (param na métrica, no máximo botam log). Eu já tenho dois terços no ar. Fechar com Tempo + correlação me põe à frente da média do mercado, e é história concreta pra contar em entrevista: "subi a stack o11y correlacionada num ambiente de produção real".

## 7. Detector de entrevista

As perguntas que separam quem entende observabilidade de quem decorou "o que é um span":

1. **"Qual a diferença entre monitoring e observability?"** — Monitoring responde falhas que você *previu* (known-unknowns, dashboard pronto). Observability responde perguntas *novas* sobre estado interno sem subir código (unknown-unknowns). Quem responde "observability é monitoring mais avançado" não entendeu.
2. **"Baggage é um dos pilares/sinais?"** — Não. É context propagation: chave-valor que viaja na request pra enriquecer os *outros* sinais, e não vira atributo sem você escrever explicitamente. Pega quem decorou lista.
3. **"Tenho métrica, log e trace nos dashboards. Isso é observabilidade?"** — Não, não sem **correlação** (trace_id, exemplars, semantic conventions). Três sinais em silo = três monitorings.
4. **"Por que não amostrar 100% dos traces? Head vs tail?"** — Custo. Head decide no início (barato, cego pro raro); tail decide no fim vendo o trace todo (pega erro/lentidão, mas é caro e stateful).
5. **"Qual sinal você olha primeiro numa latência intermitente e por quê?"** — Trace, pra localizar o span/serviço; depois métrica pro escopo, log pra causa. Mostra que sei *pra que serve* cada um, não só o que é.

---

*Próximo: Tema 2, Traces & Spans a fundo — onde o "caminho da request" que antecipei aqui vira o assunto oficial: span, relação parent/child, span context, atributos e o que é propagado entre serviços.*
