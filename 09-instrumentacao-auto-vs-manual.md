# Tema 9 · Instrumentation (auto vs manual)

> `TL;DR:` Duas formas de gerar telemetria: **manual** (code-based, você escreve `tracer.start_span()` direto, controle fino, mais trabalho) e **zero-code/automática** (agent/lib injeta instrumentação sem tocar no seu código, rápido de ligar, granularidade menor). Não é "ou/ou" — na prática produção roda as duas juntas. E dentro de qualquer uma delas, a escolha de **Span Processor** (Simple vs Batch) decide se a exportação trava o request do seu usuário ou roda em background.

---

## 1. A ideia antes do nome: duas perguntas diferentes — "o que o request faz por fora" vs "o que a minha lógica faz por dentro"

Zero-code cobre "o que acontece nas bordas da aplicação" — chamada HTTP entrando, query no banco, chamada pra fila. Isso é genérico o bastante pra uma lib de instrumentação de framework/driver cobrir sem saber nada do seu domínio. Manual cobre o que só você sabe que importa: "esse cálculo de preço com desconto aplicado", "essa decisão de fallback pro cache secundário" — lógica de negócio que nenhuma lib genérica adivinha.

> "Code-based instrumentation... oferece visão mais profunda e telemetria rica da aplicação em si."
> "Zero-code... são ótimas pra começar, ou quando você não consegue modificar a aplicação — fornecem telemetria rica de bibliotecas que você usa e/ou do ambiente onde sua aplicação roda."

Na prática, produção madura usa as duas ao mesmo tempo: zero-code te dá a "casca" (todo HTTP in/out, toda query) de graça, e manual preenche os buracos de negócio que a casca não cobre.

## 2. Instrumentation Libraries: zero-code, mas por biblioteca específica

"Zero-code" não é mágica genérica — é, na prática, uma **lib de instrumentação por framework/driver** (o exemplo mais comum: agente Java que faz bytecode weaving em runtime, ou `opentelemetry-instrument` no Python fazendo monkey-patch nas libs conhecidas — Flask, requests, psycopg2, etc). Cada combinação de linguagem+framework precisa da lib específica existir e estar mantida; não é "ligou uma flag e tudo vira span automaticamente" — é "ligou uma flag e tudo que tem lib de instrumentação conhecida vira span automaticamente". Framework novo, driver exótico, sem lib pronta = zero-code não cobre, cai pra manual.

## 3. Span Processor: o que decide se exportar trava seu request

Depois que um span termina, alguém precisa decidir **quando** mandar isso pro exportador. Essa é a função do Span Processor — e a spec define dois built-in com comportamento bem diferente:

**SimpleSpanProcessor** — "passes finished spans to the configured SpanExporter, as soon as they are finished. The processor MUST synchronize calls to Span Exporter's Export." Ou seja: span terminou → chama o exporter **ali mesmo, sincronamente**.

**BatchSpanProcessor** — "creates batches of finished spans and passes the export-friendly span data representations to the configured SpanExporter." Exporta quando o intervalo agendado passa, quando a fila atinge um tamanho máximo, ou quando alguém força um flush.

Pensa comigo antes de eu te dar a resposta: se `SimpleSpanProcessor` chama o exporter **sincronamente** no momento em que o span termina, e o exporter faz uma chamada de rede pro Collector/backend (Tema 10, OTLP) — o que acontece com a latência da request do seu usuário se esse exporter demorar, ou se a rede engasgar? Isso te lembra alguma coisa do Tema 8, seção 1?

### Resolução

É exatamente o mesmo problema do "SDK exportando direto pro backend sem Collector" — só que aqui, um nível abaixo: `SimpleSpanProcessor` faz a chamada de exportação **na hot path**, bloqueando o código da sua aplicação até a chamada de rede terminar (ou falhar). Em produção isso é quase sempre a escolha errada — um soluço de rede vira latência extra visível pro usuário final, na *sua* aplicação, não só no Collector. `BatchSpanProcessor` desacopla isso: o span vai pra uma fila em memória e é exportado em lote, em background, sem o código de negócio esperar a chamada de rede terminar. Regra prática: `Simple` é pra debug local/teste (você quer ver o span aparecer *imediatamente* no console); `Batch` é o default de produção, sempre.

## 4. Exportador: mesma peça do Tema 8 e 10, só que na ponta do SDK

O exporter aqui é a mesma peça de sempre (Tema 8 — troca de backend sem reescrever instrumentação; Tema 10 — fala OTLP). A diferença é só *onde* ele roda: pode ser configurado direto no SDK da aplicação (exportando pro Collector), ou — como visto no Tema 8 — o Collector é que mantém seu próprio exporter pro backend final. É a mesma abstração (`Export()`), reaparecendo em dois pontos da cadeia.

## 5. Escolhendo a abordagem certa

| Situação | Abordagem |
|---|---|
| Serviço novo, framework popular, quer ver telemetria rodando hoje | Zero-code primeiro |
| Não pode tocar no código (binário de terceiro, legado sem manutenção) | Zero-code é a única opção |
| Precisa de span/attribute que reflita uma decisão de negócio específica | Manual, sempre — nenhuma lib genérica sabe disso |
| Framework/driver exótico sem lib de instrumentação mantida | Manual, por falta de opção |
| Produção madura, de forma geral | As duas juntas — zero-code de base + manual nos pontos de negócio que importam |

## 6. Conexão com meu stack

Em Python/Go rodando em Kubernetes, o caminho mais rápido pra começar é `opentelemetry-instrument` (Python) via variável de ambiente no manifest, sem tocar em código — dá o esqueleto de spans HTTP/DB de graça. O trabalho de verdade entra depois: escolher os 3-4 pontos de lógica de negócio que merecem span manual com attribute customizado (`wh1.*`, seguindo a regra de namespace do Tema 7), e garantir que o processor em produção é `Batch`, nunca `Simple`.

## 7. O que vira pergunta de entrevista

- "Zero-code cobre tudo, então por que ainda escrever instrumentação manual?" → zero-code só instrumenta o que uma lib genérica reconhece (framework/driver); decisão de negócio específica só manual cobre.
- "Qual span processor você usaria em produção e por quê?" → `BatchSpanProcessor` — `Simple` exporta sincronamente na hot path, um soluço de rede vira latência visível pro usuário.
- "Onde entra o exporter nessa cadeia?" → é a mesma peça vendor-neutral do Tema 8/10, só que pode rodar tanto no SDK da aplicação quanto no Collector — sempre falando OTLP.

Ref: https://opentelemetry.io/docs/concepts/instrumentation/ · https://opentelemetry.io/docs/specs/otel/trace/sdk/#span-processor
