# Prep Course – OpenTelemetry Certified Associate (OTCA) Certification

> Fonte: [KodeKloud](https://learn.kodekloud.com/learn/courses/prep-course-opentelemetry-certified-associate-certification-otca)
> Instrutores: Amrith Raj (Lead Solutions Architect), Sanjeev Thiyagarajan (Network Engineer, Trainer)
> Nível: Beginner · Duração: 18.82h · **15 módulos · 182 aulas · 17.7h de vídeo · 13 labs · 26 quizzes**

## Sumário dos módulos

| # | Módulo | Aulas |
|---|--------|-------|
| 1 | Course Introduction | 3 |
| 2 | Observability Core Concepts | 6 |
| 3 | OpenTelemetry Core Concepts | 15 |
| 4 | Span Anatomy and Context Propagation | 25 |
| 5 | Instrumentation | 31 |
| 6 | Metrics Data Model | 15 |
| 7 | Recording Measurements | 11 |
| 8 | Logs | 10 |
| 9 | OTel Collector Foundations | 9 |
| 10 | OTel Collector Core Components | 19 |
| 11 | OpenTelemetry in Kubernetes | 10 |
| 12 | Transforming Telemetry – Pipeline Processing | 13 |
| 13 | OpenTelemetry Collector: Debugging, Operations, and Scaling | 11 |
| 14 | Extended Topics | 2 |
| 15 | Mock Exams | 2 |
| | **Total** | **182** |

---

## 1. Course Introduction (3 aulas)

- [ ] Introduction — 03:59
- [ ] Bob – the Software Doctor – and Mysterious Application Ailments — 03:05
- [ ] How to Reach Out to KodeKloud and Engage with the Community

## 2. Observability Core Concepts (6 aulas)

- [ ] How Applications Work in Distributed Computing — 03:58
- [ ] Introduction to Observability — 11:44
- [ ] Quiz: Introduction to Observability
- [ ] Monitoring and Observability — 05:43
- [ ] Quiz: Monitoring vs Observability
- [ ] Before OpenTelemetry: Why Standardization Was the Missing Piece — 10:08

## 3. OpenTelemetry Core Concepts (15 aulas)

- [ ] OpenTelemetry – Introduction — 09:10
- [ ] OpenTelemetry – Goals, Mission, Vision, and Values — 06:40
- [ ] Standards and Specification — 10:43
- [ ] Semantic Conventions — 08:30
- [ ] Semantic Conventions – Guidelines — 10:42
- [ ] Main Components of OpenTelemetry — 04:08
- [ ] OpenTelemetry End-to-End Architecture — 06:08
- [ ] OpenTelemetry API — 23:21
- [ ] OpenTelemetry Client Design Principles — 12:07
- [ ] OpenTelemetry Client Architecture — 08:22
- [ ] OpenTelemetry Specification Status — 03:32
- [ ] OpenTelemetry Protocol - OTLP — 14:41
- [ ] OTLP - Transport Over gRPC and HTTP — 10:23
- [ ] Quiz: OTLP Transport Mechanisms
- [ ] Quiz: OpenTelemetry Core Concepts

## 4. Span Anatomy and Context Propagation (25 aulas)

- [ ] Distributed Trace – Introduction — 08:59
- [ ] OpenTelemetry Spans — 07:31
- [ ] Quiz: Distributed Traces & Span Concepts
- [ ] Span Name and Context — 16:35
- [ ] Quiz: Span Name and Context
- [ ] Span Kind — 06:22
- [ ] Quiz: Span Kind
- [ ] Span Links — 04:54
- [ ] Quiz: Span Links
- [ ] Span Status and Exceptions — 08:11
- [ ] Quiz: Span Status
- [ ] Span Timings — 04:03
- [ ] Span Events — 04:01
- [ ] Span Attributes — 07:56
- [ ] Span Events vs Span Attributes — 03:09
- [ ] Quiz: Spans: Timings, Events, Attributes
- [ ] Span Resource — 10:33
- [ ] Quiz: Span Resource
- [ ] OpenTelemetry Baggage — 08:33
- [ ] Quiz: OpenTelemetry Baggage
- [ ] Context Propagation — 25:05
- [ ] Quiz: Context Propagation
- [ ] Span Anatomy Summary — 06:57
- [ ] Demo: Tracing Overview — 14:10
- [ ] Lab: Tracing Overview

## 5. Instrumentation (31 aulas)

- [ ] OpenTelemetry Instrumentation Approaches — 03:53
- [ ] Code-Based (Manual) Instrumentation and Tracing API – Introduction — 20:35
- [ ] Quiz: Instrumentation & Code-Based Instrumentation
- [ ] Demo: Instrumenting Application - Configuring OpenTelemetry — 10:35
- [ ] Demo: Instrumenting Application - Creating the First Span — 11:48
- [ ] Span Processors and Exporters — 08:13
- [ ] Demo: Span Processors — 02:23
- [ ] Demo: Exporters — 04:39
- [ ] Quiz: Span Processors and Exporters
- [ ] Demo: Resource Attributes — 01:21
- [ ] Demo: Instrumenting API — 06:34
- [ ] Demo: Connecting Two Services — 07:16
- [ ] Demo: Propagating Context — 08:36
- [ ] Demo: Events — 03:04
- [ ] Demo: Exceptions — 03:51
- [ ] Demo: Status Codes — 03:07
- [ ] Span Sampling — 08:18
- [ ] Demo: Sampling — 03:01
- [ ] Quiz: Span Sampling
- [ ] Lab: Manual Instrumentation
- [ ] Instrumentation Libraries — 13:20
- [ ] Zero Code/Automatic Instrumentation — 07:46
- [ ] Quiz: Zero-Code (Automatic) Instrumentation
- [ ] Zero-Code Instrumentation in Java — 11:49
- [ ] Quiz: Zero-Code Techniques in Java
- [ ] Zero-Code Instrumentation in Python — 10:45
- [ ] Demo: Zero-Code Techniques in Python — 16:08
- [ ] Lab: Auto Instrumentation
- [ ] Choosing the Right Instrumentation Approach — 04:30
- [ ] Quiz: Choosing the Right Instrumentation Approach
- [ ] Demo: Span Attributes — 02:56

## 6. Metrics Data Model (15 aulas)

- [ ] Role of Metrics in Observability — 08:42
- [ ] Quiz: Role of Metrics in Observability
- [ ] Monotonicity — 01:49
- [ ] Temporality — 03:38
- [ ] Aggregation and Histogram — 04:43
- [ ] Dimensions — 04:36
- [ ] Cardinality — 06:25
- [ ] Recap - Structural Prop of Metrics — 01:57
- [ ] Quiz: Structural Properties of Metrics
- [ ] Metrics Data Model — 08:11
- [ ] The Metrics Data Journey With OpenTelemetry — 05:34
- [ ] Quiz: The Metrics Data Journey With OpenTelemetry
- [ ] Lab: Metrics Overview
- [ ] Metric Instruments in OpenTelemetry — 08:19
- [ ] Quiz: Metric Instruments in OpenTelemetry

## 7. Recording Measurements (11 aulas)

- [ ] Metrics API and SDK — 08:33
- [ ] Metrics Pipeline — 05:13
- [ ] Metrics SDK Lifecycle APIs and Views — 04:37
- [ ] Exemplars — 05:19
- [ ] Demo: Metrics Instrumentation — 06:40
- [ ] Demo: Counter Metric — 15:04
- [ ] Demo: Updown Counter — 06:54
- [ ] Demo: Async Gauge — 07:59
- [ ] Demo: Histogram — 09:54
- [ ] Demo: Prometheus Exporter — 04:31
- [ ] Lab: Metrics Instrumentation

## 8. Logs (10 aulas)

- [ ] OpenTelemetry-Logging-Introduction — 10:29
- [ ] Open Telemetry Logging Components — 06:24
- [ ] Open Telemetry - Logs Data Model — 09:00
- [ ] Logs API — 04:12
- [ ] Logs SDK — 04:30
- [ ] Demo: Your First Log — 09:12
- [ ] Demo: Using Python Stdlib — 04:56
- [ ] Demo: Integrating Traces With Logs — 02:13
- [ ] Demo: Exporting Logs to Collector — 02:32
- [ ] Lab: Logging Instrumentation

## 9. OTel Collector Foundations (9 aulas)

- [ ] OTel Collector Purpose Slide Deck — 04:30
- [ ] OTel Collector Distributions — 07:32
- [ ] Demo: OTel Collector Installation — 04:08
- [ ] Demo: OTel Collector Docker Installation — 04:45
- [ ] Lab: OTel Collector Deployment
- [ ] OTel Collector Anatomy — 02:17
- [ ] Understanding config.yaml in OpenTelemetry Collector — 07:04
- [ ] Demo: OpenTelemetry Collector Configuration — 12:19
- [ ] Lab: OpenTelemetry Collector Configuration

## 10. OTel Collector Core Components (19 aulas)

- [ ] Collector as an Agent — 02:56
- [ ] OpenTelemetry Collector: Receivers — 14:02
- [ ] OpenTelemetry Collector: Processors — 16:32
- [ ] OpenTelemetry Collector: Exporters — 01:19
- [ ] OTel Col: OTLP Exporter and Resiliency — 06:07
- [ ] OTel Col: Contrib Exporters — 05:16
- [ ] OTel Col: Load Balancing Exporter — 06:22
- [ ] OTel Col: Exporter Patterns, Reliability, and Best Practices — 02:39
- [ ] Demo: OTel Col Jaeger Exporter — 05:38
- [ ] Demo: OTel Col Processors — 05:19
- [ ] Demo: OTel Col Attribute Processor — 03:43
- [ ] Demo: OTel Col Metrics — 06:40
- [ ] Demo: OTel Col Filter — 07:20
- [ ] Demo: OTel Col Prometheus Exporter — 04:32
- [ ] Lab: OpenTelemetry Collector Configuring Receivers Exporters
- [ ] OTel Col Connectors — 14:48
- [ ] Demo: OTel Col Connectors — 02:48
- [ ] Service and Pipelines Explained — 10:33
- [ ] Lab: Processors and Connectors

## 11. OpenTelemetry in Kubernetes (10 aulas)

- [ ] OpenTelemetry in Kubernetes — 01:58
- [ ] OpenTelemetry Kubernetes Operator — 02:58
- [ ] Auto-Instrumentation Using the OpenTelemetry Operator — 07:29
- [ ] Important Receivers for Kubernetes — 02:41
- [ ] OpenTelemetry Collector Deployment Modes in Kubernetes — 09:35
- [ ] Demo: OpenTelemetry Collector k8s — 24:26
- [ ] Demo: OpenTelemetry Operator — 10:51
- [ ] Demo: OpenTelemetry Operator Auto-Instrumentation — 05:29
- [ ] Lab: OpenTelemetry on Kubernetes
- [ ] Lab: OpenTelemetry k8s Operator

## 12. Transforming Telemetry – Pipeline Processing (13 aulas)

- [ ] OTTL Basics — 10:11
- [ ] Demo: OTTL Filter — 16:47
- [ ] Demo: OTTL Transform - Part 1 — 11:49
- [ ] Demo: OTTL Transform - Part 2 — 10:22
- [ ] Lab - OTTL
- [ ] Sampling Strategies — 03:08
- [ ] Head Sampling — 03:18
- [ ] Tail Sampling — 08:58
- [ ] Demo: Sampling Strategies — 16:48
- [ ] Schemas: Why Schemas Matter — 04:06
- [ ] Schema Fundamentals — 07:02
- [ ] Schema Translation Rules by Signal Type — 11:35
- [ ] Check and Reflect: Schema Summary — 03:43

## 13. OpenTelemetry Collector: Debugging, Operations, and Scaling (11 aulas)

- [ ] Internal Logs of the Collector — 05:15
- [ ] Debug Exporter — 03:49
- [ ] Demo: Collector Internal Logs and Debug Exporter — 11:31
- [ ] Internal Metrics of the Collector — 06:43
- [ ] Extensions: healthcheck — 01:21
- [ ] Extensions: zPages — 05:52
- [ ] Extensions: pprof — 02:35
- [ ] Recap: When to Use What — 02:29
- [ ] Demo: OTel Collector Extensions — 08:34
- [ ] Scaling the OpenTelemetry Collector — 08:20
- [ ] Demo: Internal Metrics of the Collector — 04:00

## 14. Extended Topics (2 aulas)

- [ ] SLIs and SLOs With OpenTelemetry — 08:52
- [ ] Quiz: SLIs and SLOs With OpenTelemetry

## 15. Mock Exams (2 aulas)

- [ ] Mock Exam - 1
- [ ] Mock Exam - 2

---

### Descrição de cada módulo (do texto oficial do curso)

**Observability Core Concepts** — Build a solid foundation in observability by understanding how modern distributed applications work, the difference between monitoring and observability, and why standardization through OpenTelemetry was the missing piece in modern system reliability.

**OpenTelemetry Core Concepts** — Explore OpenTelemetry's goals, mission, and vision while diving deep into its architecture, semantic conventions, core components, API, and specification standards. Understand how the OpenTelemetry protocol (OTLP) enables seamless interoperability and gain insights into the end-to-end telemetry pipeline.

**Span Anatomy and Context Propagation** — Master distributed tracing fundamentals by understanding spans, span kinds, context propagation, and attributes. Learn to connect multiple services, propagate context across them, and analyze traces using real-world demos and labs.

**Instrumentation** — Discover how to instrument applications manually and automatically to generate telemetry data. Learn about span processors, exporters, resource attributes, and sampling strategies through practical demonstrations in multiple programming languages, including Python and Java.

**Metrics Data Model** — Understand the role of metrics in observability and learn about the structural properties of metrics such as monotonicity, temporality, and aggregation. Explore the OpenTelemetry metrics data model, instruments, and data flow to gain complete control over metric collection.

**Recording Measurements** — Get hands-on with the OpenTelemetry Metrics API and SDK. Learn about metric lifecycles, views, and exemplars while recording real-time measurements. Practice with demos and labs to build, visualize, and export metrics using tools like Prometheus.

**Logs** — Learn how OpenTelemetry handles logging across distributed systems. Explore the logging data model, APIs, SDKs, and integrations with tracing. Through demos and labs, you'll instrument logs, export them via the OpenTelemetry Collector, and correlate them with trace data for full-stack observability.

**OTel Collector Foundations** — Understand the purpose and architecture of the OpenTelemetry Collector. Learn installation methods, configuration fundamentals, and distribution models. Through guided demos and labs, deploy and configure the Collector on various environments including Docker and Kubernetes.

**OTel Collector Core Components** — Deep dive into the Collector's key building blocks: receivers, processors, exporters, and connectors. Learn to configure and optimize telemetry pipelines, apply best practices for resiliency, and integrate with backends like Jaeger and Prometheus through detailed demos and labs.

**OpenTelemetry in Kubernetes** — Discover how to deploy and manage OpenTelemetry within Kubernetes environments. Learn about the OpenTelemetry Operator, auto-instrumentation techniques, key Kubernetes receivers, and Collector deployment modes. Practice these concepts through interactive demos and hands-on Kubernetes labs.

**Transforming Telemetry – Pipeline Processing** — Gain expertise in transforming and filtering telemetry data using the OpenTelemetry Transformation Language (OTTL). Learn to apply filtering, attribute transformation, and sampling strategies like head and tail sampling. Understand schema fundamentals and translation rules to maintain data consistency across observability systems.

**OpenTelemetry Collector: Debugging, Operations and Scaling** — Learn to operate and scale the OpenTelemetry Collector in production. Understand debugging techniques, internal diagnostics, health checks, and performance profiling using zPages and pprof. Explore strategies for horizontal scaling, resilience, and troubleshooting telemetry pipelines efficiently.

**Mock Exams and Exam Readiness** — Mock exams that mirror the structure, coverage, and difficulty of the official OTCA certification, to reinforce knowledge, assess readiness, and boost confidence for the real exam.
