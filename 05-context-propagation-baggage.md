# Tema 5 · Context propagation & Baggage

> `TL;DR:` Context propagation é o mecanismo que carrega o Span Context (Tema 2) de um processo pro outro — sem ele, cada serviço criaria uma trace nova e isolada, e nunca teria "distributed" em distributed tracing. Funciona em dois passos: **inject** (serializa o contexto num carrier, ex: header HTTP) do lado de quem chama, **extract** (lê de volta) do lado de quem recebe. O padrão da indústria é o W3C Trace Context, transportado no header `traceparent`. Baggage (Tema 1) usa o mesmo mecanismo de inject/extract, só que carrega dado de negócio arbitrário em vez do Span Context.

---

## 1. A ideia antes do nome: o problema é a rede não ter memória

Dentro de um processo só, propagar "quem é o span pai" é trivial — é só uma variável em memória, ou uma thread-local. O problema aparece na fronteira entre processos: quando o Serviço A chama o Serviço B por HTTP, a rede não carrega automaticamente "esse span pertence a essa trace, e o pai dele é aquele span ali". Sem um mecanismo explícito, B recebe a request pelada e, na melhor das hipóteses, começa uma trace **nova**, desconectada da de A.

Definição oficial de Context:

> "Context is an object that contains the information for the sending and receiving service, or execution unit, to correlate one signal with another."

Concretamente: quando A chama B, o contexto carrega o Trace ID e o Span ID de A — B usa isso pra criar seu próprio span já pertencendo à **mesma** trace, com o span de A como pai. É esse mecanismo, repetido em cada fronteira de rede, que constrói a árvore inteira do Tema 2 através de N serviços arbitrariamente distribuídos.

## 2. Propagator: inject de um lado, extract do outro

> "Propagation is the mechanism that moves context between services and processes."

Dois passos, sempre no par emissor/receptor:

- **Inject** — do lado de quem chama: serializa o contexto atual **dentro** do carrier da chamada. "Injected into the carrier, for example, into the headers of an HTTP request."
- **Extract** — do lado de quem recebe: lê o contexto de volta do carrier. "Extracted from the carrier. Again, in the case of HTTP, this is retrieved from the headers."

Isso normalmente é **transparente** — a lib de instrumentação HTTP do seu framework já faz inject/extract sozinha, você não escreve esse código manualmente na maioria dos casos. Onde isso vira problema visível: quando você tem uma chamada que a lib de instrumentação **não** cobre automaticamente (ex: uma fila customizada, um protocolo binário interno) — aí o inject/extract é código que alguém precisa escrever à mão, ou o trace quebra exatamente naquela fronteira.

## 3. O formato padrão: W3C Trace Context, header `traceparent`

O propagador default hoje é a especificação W3C (não é invenção do OTel — é um padrão da indústria que o OTel adotou). Formato:

```
<version>-<trace-id>-<parent-id>-<trace-flags>
```

Exemplo real: `00-a0892f3577b34da6a3ce929d0e0e4736-f03067aa0ba902b7-01`

- `00` — versão do formato.
- `a0892f3577b34da6a3ce929d0e0e4736` — Trace ID (o mesmo em toda a árvore).
- `f03067aa0ba902b7` — Parent ID (o Span ID de quem tá chamando — vira o `parent` do span que B vai criar).
- `01` — trace flags (aqui, indicando sampled).

Isso viaja como header HTTP normal (`traceparent: ...`), texto puro, sem exigir nada especial do transporte — é por isso que qualquer proxy/load balancer no meio, mesmo sem entender OTel, deixa passar sem problema (desde que não filtre headers custom).

## 4. Baggage usa o mesmo mecanismo, carrega outra coisa

O Tema 1 já cobriu o que baggage **é** (dicionário chave-valor de negócio, não um sinal) e o cuidado de segurança (nunca PII/segredo, viaja em texto claro). O ponto que fecha aqui: baggage usa **o mesmo par inject/extract**, o mesmo conceito de propagador — só que o padrão W3C pra ele é outro header (`baggage`), separado do `traceparent`. Span Context (obrigatório, é o que faz a trace existir) e Baggage (opcional, dado de negócio) trafegam lado a lado, mas são propagadores distintos, cada um com seu formato.

## 5. Por que isso é o que faz "distributed" em distributed tracing

> "[Propagation enables] traces to build causal information about a system across services that are arbitrarily distributed across process and network boundaries."

Sem propagação de contexto, cada serviço no caminho de uma request produziria uma trace isolada e sem relação com as outras — você teria N traces fragmentadas em vez de uma árvore coerente. Propagação é o que faz a promessa central do Tema 1 (observabilidade via correlação, não pilar isolado) valer atravessando fronteira de processo, não só dentro de um serviço só.

## 6. Conexão com meu stack

Em Kubernetes com múltiplos serviços, o ponto de falha mais comum de tracing não é o SDK — é uma fronteira de rede que o inject/extract automático não cobre: um `kubectl exec` chamando algo direto, um job assíncrono lançado sem carregar o carrier, um gRPC interceptor mal configurado. Quando um trace no Tempo "quebra" no meio — vira dois traces separados em vez de um — a primeira suspeita é sempre "que fronteira não tá propagando o `traceparent`".

## 7. O que vira pergunta de entrevista

- "O que faz um trace ser 'distribuído' de verdade?" → propagação de contexto entre processos, não só instrumentação dentro de cada um isoladamente.
- "Onde o Span Context viaja numa chamada HTTP?" → no header `traceparent`, padrão W3C Trace Context.
- "Baggage e Span Context são propagados do mesmo jeito?" → mesmo mecanismo (inject/extract), headers W3C diferentes (`traceparent` vs `baggage`) — um é obrigatório pra trace existir, o outro é dado de negócio opcional.

Ref: https://opentelemetry.io/docs/concepts/context-propagation/
