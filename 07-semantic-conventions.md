# Tema 7 · Semantic conventions

> `TL;DR:` Semantic conventions são o dicionário comum de nomes que o OTel define pra spans, métricas, logs e atributos. Sem isso, cada instrumentação inventa seu próprio vocabulário (`http_method` vs `verb` vs `method_name`) e nada correlaciona entre serviços — mesmo os dois falando o mesmo protocolo (OTLP). Regra prática: nunca invente nome de atributo pra algo que a convenção já cobre.

---

## 1. A ideia antes do nome: protocolo comum não é vocabulário comum

O Tema 10 (OTLP) resolve "como os dados trafegam" — formato, transporte. Mas dois serviços podem falar OTLP perfeitamente e ainda serem incompreensíveis um pro outro: se um chama o atributo do método HTTP de `http.method` e o outro de `verb`, seu dashboard não sabe cruzar os dois. É como duas pessoas usando o mesmo protocolo de telefonia — a ligação conecta, o áudio chega limpo — mas falando línguas diferentes. O canal funciona, a comunicação não.

Definição oficial:

> "Semantic Conventions specify common names for different kinds of operations and data. The benefit to using Semantic Conventions is in following a common naming scheme that can be standardized across a codebase, libraries, and platforms."

Exemplo concreto: um cliente HTTP instrumentado via semantic conventions sempre produz `http.request.method`, `url.full`, `http.response.status_code`, `server.address` — não importa se o time A escreveu em Go e o time B em Python. Dashboard, alerta e query filtrando por `http.response.status_code` funcionam nos dois sem gambiarra de normalização.

## 2. Onde a convenção mora: namespace por domínio, atravessando os sinais

As convenções são organizadas por **domínio** (namespace com prefixo): `http.*`, `db.*`, `rpc.*`, `messaging.*`, `faas.*`, `graphql.*`. Cada namespace define nome do atributo, tipo, significado e valores válidos — e também nomes padronizados de span/métrica pra aquele tipo de operação.

E elas atravessam os **sinais**: trace, metric, log, event, profile, resource. Um atributo de resource como `service.name` ou `k8s.pod.name` aparece igual não importa qual sinal você está olhando — é literalmente isso que viabiliza a correlação trace↔log↔métrica do Tema 1: se os três sinais concordam no nome do atributo, dá pra pivotar de um pro outro na mesma query.

Se você precisar de um atributo custom que a convenção não cobre, a regra oficial é: coloque no seu próprio namespace (ex: `wh1.tenant.id`), nunca solte um atributo sem prefixo — evita colisão com um nome que o OTel venha a padronizar depois.

## 3. O ponto chato: a convenção muda de versão, e sua telemetria antiga não pode quebrar

Nomes de atributo **mudam** conforme a spec evolui — por exemplo, uma versão antiga usava `http.method`, a atual usa `http.request.method`. Pensa nisso: se você tem um dashboard no Grafana com uma query hardcoded em `http.method`, e amanhã atualiza a lib de instrumentação pra uma versão que já emite `http.request.method`, o que acontece com o painel? Não dá erro, não quebra o deploy — o painel só fica **vazio**, silenciosamente, porque a query não bate com o nome novo. Como o OTel evita que isso force todo mundo a nunca atualizar (medo de quebrar dashboard) ou a reescrever toda query a cada release de semconv? Pensa em pelo menos uma forma de o dado carregado saber "de qual versão de convenção eu vim" antes de eu resolver.

### Resolução: Telemetry Schemas e `schema_url`

Cada lote de telemetria carrega um `schema_url`, que aponta pra um arquivo de schema descrevendo a história de transformações daquela convenção — basicamente um diff versionado tipo "atributo X na v1 == atributo Y na v2". Uma ferramenta de análise (backend, Grafana, o que for) que suporta schema pode ler o `schema_url`, descobrir de qual versão aquele dado veio, e aplicar a transformação registrada pra normalizar dado antigo e novo na mesma query.

> "Changes to semantic conventions ... are allowed, provided that the changes can be described by schema files."

Ou seja: renomear atributo não é quebra de contrato — é uma mudança **esperada**, desde que documentada em schema. Na prática, se seu backend não processa schema_url (nem todos processam), a defesa é mais chata: acompanhar o changelog de semantic conventions quando você bumpa versão de lib de instrumentação, não só o changelog da lib em si.

## 4. Maturidade: Development → Stable → Deprecated → Removed

- **Development** — pode ter breaking change a qualquer momento, sem aviso prévio. Não crie dependência de longo prazo (alerta crítico, SLO) em cima de um atributo nesse estágio.
- **Stable** — mudança incompatível só acontece com bump de versão *major*. Aqui sim dá pra depender pra valer.
- **Deprecated** — só acontece depois que o substituto já é Stable.
- **Removed** — exige bump major.

Regra prática pra quando eu for instrumentar algo: antes de fincar um atributo específico num alerta ou dashboard importante, checa se ele já é Stable ou ainda tá em Development — atributo experimental muda o nome debaixo do seu pé.

## 5. Conexão com meu stack

Prometheus (relabeling), Loki (LogQL) e Grafana (dashboards salvos) todos travam em nome de atributo/label. Isso significa que uma query salva no Grafana carrega o mesmo risco do exemplo da seção 3 — semconv renomeia, dashboard fica mudo, sem erro nenhum pra chamar atenção. Hábito prático: quando bumpar versão de lib de instrumentação (auto-instrumentation do Python/Java, por exemplo), checa o changelog de semantic conventions daquele release, não só o changelog da lib.

## 6. O que vira pergunta de entrevista

- "Por que semantic conventions existem se já existe OTLP?" → protocolo padroniza **transporte**, convenção padroniza **vocabulário**; sem os dois juntos não tem interoperabilidade de verdade entre instrumentações independentes.
- "O que acontece quando um atributo muda de nome numa versão nova da convenção?" → não é breaking change não-documentado: vem descrito num schema file, referenciado via `schema_url`, pra ferramenta normalizar dado antigo e novo.
- "Como você nomearia um atributo custom que a convenção não cobre?" → no seu próprio namespace prefixado, nunca solto — evita colisão com convenção futura.

Refs: https://opentelemetry.io/docs/specs/semconv/ · https://opentelemetry.io/docs/specs/otel/versioning-and-stability/
