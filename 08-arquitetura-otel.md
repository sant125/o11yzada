# Tema 8 · Arquitetura OTel (API, SDK, Collector)

> `TL;DR:` Sem um padrão, só sobram duas saídas ruins: **lock-in** (instrumenta o código direto com o SDK proprietário de um vendor — Datadog, New Relic, sei lá — e migrar de ferramenta vira reescrever a instrumentação inteira) ou **silos** (cada time/sinal usa uma ferramenta diferente, sem convenção comum de nomes, e não dá pra correlacionar nada entre elas). A sacada do OTel foi separar **como instrumento** de **pra onde os dados vão**: API/SDK vendor-neutral pra instrumentar uma vez, e o Collector decide o destino via exporter — troca de backend sem tocar no código instrumentado.

---

## 0. Missão, visão e valores (oficial) — fechando o "porquê" antes da arquitetura

Antes de entrar na arquitetura, o texto oficial do projeto resume em uma frase o motivo dele existir:

> **Missão:** "enable effective observability by making high-quality, portable telemetry ubiquitous."

A visão se desdobra em 5 apostas de que telemetria deve ser: **fácil** (defaults bons, baixo tempo-pra-valor), **universal** (mesmo protocolo/convenção entre linguagens e sinais), **vendor-neutral** (sem lock-in — exatamente a Cilada 1 da seção 1), **loosely coupled** (usa só o componente que precisa, não o projeto inteiro) e **built-in** (parte da stack, não um apêndice).

Valores de engenharia: compatibilidade, estabilidade, resiliência, performance, auto-observabilidade (o Collector saber expor métricas/logs de si mesmo — Tema 13). Valores de comunidade: agir no interesse do projeto, declarar conflito de interesse, presumir boa intenção, discordar com respeito.

Pra entrevista, a frase que interessa é a missão: **telemetria de alta qualidade e portátil, onipresente** — "onipresente" resume "built-in" e "fácil" ao mesmo tempo.

Ref: https://opentelemetry.io/community/mission/

## 1. A sacada: instrumentar não devia ser acoplado ao backend

Antes de existir um padrão, instrumentar um sistema te jogava em uma de duas ciladas:

**Cilada 1 — Vendor lock-in.** Você instrumenta o código direto com o SDK proprietário do vendor (`ddtrace` do Datadog, o agent do New Relic, o SDK do Dynatrace). Funciona, e até funciona bem — mas cada `tracer.start_span()`, cada decorator, cada nome de atributo é **daquele** vendor. O dia que o contrato fica caro, ou a empresa decide migrar, não é "trocar um endpoint" — é **reescrever toda a instrumentação** espalhada em dezenas de serviços. Você não é refém da qualidade da ferramenta, é refém do custo de sair dela.

**Cilada 2 — Silos entre ferramentas.** Pra fugir do lock-in, um jeito ingênuo é: cada time escolhe a ferramenta que quiser pra cada sinal — um usa Zipkin pra trace, outro usa StatsD pra métrica, outro joga log cru num ELK. Sem padrão comum de nomenclatura (o que no Tema 1 chamei de semantic conventions), o `trace_id` de um não bate com o `trace_id` do outro, um chama de `http_method` e outro de `verb`. Você tem *n* ferramentas, cada uma cega pras outras — voltando pro problema do Tema 1, seção 4: pilar isolado não é observabilidade, e aqui nem dá pra amarrar o fio porque nem o vocabulário é o mesmo.

A sacada do OTel foi perceber que o problema real não é "qual ferramenta", é que **instrumentação e backend estavam grudados**. A separação que ele impõe:

- **API + SDK** — camada vendor-neutral que você usa pra instrumentar o código. Sempre a mesma, não importa o backend.
- **OTLP** — o protocolo de transporte, também neutro.
- **Collector** — o ponto onde você decide o destino (exporter). Trocar de backend é trocar uma linha de config no Collector, não recompilar e reescrever instrumentação em 40 serviços.

Resumindo em uma frase pra entrevista: **OTel não compete com Datadog/New Relic/Tempo — ele compete com o SDK proprietário deles.** O backend continua sendo escolha sua; o que muda é que a instrumentação para de ser refém dessa escolha.

> **Antes de eu resolver isso pra você:** o SDK do OTel já fala OTLP nativamente — dava pra exportar direto do serviço pro backend, sem Collector no meio. Quase ninguém faz assim em produção. Por quê? Pensa em pelo menos dois motivos antes de eu chegar no Tema 11 (Collector a fundo). Dica: pensa no que acontece quando o backend cai, e no que acontece quando você precisa mudar o destino de 40 serviços de uma vez.

### Resolução: por que Collector, e não SDK → backend direto

**Motivo 1 — buffer/broker que reduz a janela de perda.** Se o serviço exporta OTLP direto pro backend e o backend cai, os dados daquela janela se perdem — o SDK só segura um buffer pequeno em memória antes de descartar. O Collector no meio absorve esse soluço: recebe, enfileira, tenta de novo com backoff, e só entrega quando o backend volta.

Detalhe importante: a fila padrão do Collector (`sending_queue`) é **em memória**. Se o próprio Collector reiniciar ou crashar com dados na fila, eles somem — não tem durabilidade por padrão. WAL de verdade só existe se você liga **persistent queue** (extension `file_storage`), que grava em disco e sobrevive a restart do Collector. Ou seja: Collector sem persistent queue já **reduz muito** a janela de perda (o backend cair não derruba nada), mas só vira **garantia sólida contra perda** com a fila persistente configurada.

Bônus que fecha o motivo: isso também tira o retry/backoff da **hot path** do teu serviço. Sem Collector, é o processo da aplicação que fica retentando contra um backend fora do ar — gastando memória/CPU dele, no meio do request. Com Collector, esse trabalho vira problema de infra, isolado do código de negócio.

**Motivo 2 — um ponto de mudança em vez de N.** Na prática o endpoint OTLP costuma ser env var (`OTEL_EXPORTER_OTLP_ENDPOINT`), não hardcoded — então trocar de backend não pede rebuild. O problema real de não ter Collector é outro: você precisaria tocar essa config em **N serviços, N deploys, N janelas de rollout coordenadas**. Com Collector, é **um lugar só** — muda o exporter na config dele, e os 40 serviços nem sabem que o destino mudou.

Resumo pra entrevista: **Collector desacopla o serviço da disponibilidade e da identidade do backend** — o primeiro motivo é sobre *quando* o backend falha, o segundo é sobre *quem* é o backend.

Ref: https://opentelemetry.io/docs/collector/architecture/ · https://opentelemetry.io/docs/collector/configuration/#persistent-queue

## 2. Exemplo prático: enriquecendo Datadog sem lock-in

Isso rende até quando o backend que você usa **é** um vendor proprietário, tipo Datadog. Duas formas de chegar lá:

- **Errado (lock-in):** instrumenta cada serviço com `dd-trace` (o SDK do Datadog). Span, atributo, tudo no vocabulário deles. Migrar pra Grafana Cloud ou Honeycomb amanhã = reescrever.
- **Certo (via OTel):** instrumenta com o **SDK do OpenTelemetry**, manda pro **Collector**, e o Collector exporta via **OTLP** pro Datadog Agent — que hoje aceita ingestão OTLP nativamente. O Datadog continua sendo o backend que você olha todo dia; só que a instrumentação não é dele. Se um dia trocar pra Tempo/Honeycomb/o que for, é **trocar o exporter no Collector**, zero mudança nos serviços.

Esse é literalmente o pitch que separa quem "usa OTel" de quem só "usa Datadog e chama de observabilidade".

## 3. Ferramentas pra ter familiaridade (linha que tô seguindo)

O ponto em comum entre todas: hoje aceitam **OTLP** como entrada, então dá pra trocar de uma pra outra sem reinstrumentar.

- **Datadog** — vendor líder de mercado, forte candidato a aparecer em entrevista/trabalho; aceita OTLP via Agent.
- **Grafana Tempo** — já é minha linha (Tema 1, seção 6), open source, backend de trace que fecha Prometheus+Loki+Tempo.
- **Jaeger** — o clássico open source de trace, bom de conhecer mesmo não sendo o que eu vou rodar.
- **Honeycomb** — referência em observabilidade orientada a *high-cardinality events*, vale conhecer o discurso deles.
- **New Relic / Dynatrace** — outros vendors proprietários grandes, aparecem bastante em vaga.
- **SigNoz** — alternativa open source "tudo em um" (métrica+log+trace), self-hosted, cresce em empresas que não querem pagar vendor.

## 4. API vs SDK na prática

A separação da seção 1 (API/SDK vendor-neutral vs Collector decidindo destino) tem uma segunda camada, dentro do próprio "API/SDK": **API e SDK também são pacotes separados entre si**, e essa separação é o que faz uma biblioteca de terceiros conseguir instrumentar código sem forçar todo mundo que a usa a carregar telemetria de verdade.

Definição oficial:

> "API packages consist of the cross-cutting public interfaces used for instrumentation. Any portion of an OpenTelemetry client which is imported into third-party libraries and application code is considered part of the API."

> "The SDK is the implementation of the API provided by the OpenTelemetry project. Within an application, the SDK is installed and managed by the application owner."

Analogia: a **API** é a tomada na parede — a interface física padronizada que qualquer aparelho (biblioteca) usa pra plugar, sem precisar saber se tem gerador, rede elétrica da cidade, ou nada atrás dela. O **SDK** é a fiação de verdade, instalada e ligada por quem é dono da casa (a aplicação final) — é o que decide se energia (telemetria) realmente vai fluir, e pra onde.

Consequência prática: se um driver de banco de dados, ou um client HTTP wrapper, importa só `opentelemetry-api` pra criar spans, e a aplicação que usa esse driver **nunca registra o SDK**, essas chamadas de span viram **no-op** — não dão erro, não custam quase nada, simplesmente não geram telemetria nenhuma. Só quando a aplicação final, lá no `main()`/entrypoint, importa e configura o SDK (sampler, processor, exporter — Tema 11) é que a árvore inteira de chamadas de API — o código próprio **e** o de qualquer lib de terceiros instrumentada — vira telemetria real.

Regra da especificação, direto pra quem escreve uma lib:

> "Instrumentation authors MUST NOT directly reference any SDK package of any kind, only the API."

Por que essa regra existe: se uma biblioteca importasse o SDK diretamente, ela estaria forçando toda aplicação que a usa a carregar peso de configuração de telemetria (samplers, exporters, todo o SDK) mesmo que aquela aplicação nunca vá coletar nada — e pior, prendendo a lib a uma versão/config específica de SDK que pode conflitar com o que a aplicação já usa. API sozinha resolve isso: dependência leve, opcional, e sem acoplamento.

Pra entrevista: "por que uma lib instrumentada com OTel não trava a aplicação que a usa?" — porque a lib depende só da **API**, que é no-op até alguém plugar o **SDK**; a decisão de coletar telemetria (e como) é sempre da aplicação final, nunca da biblioteca.

Ref: https://opentelemetry.io/docs/specs/otel/overview/ · https://opentelemetry.io/docs/concepts/instrumentation/#api-and-sdk

*(placeholder — próxima sessão: Collector — receivers/processors/exporters/pipelines, e onde isso se conecta com o Tema 11.)*
