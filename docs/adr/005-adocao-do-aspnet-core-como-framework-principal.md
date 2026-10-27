# ADR-005: Adoção do ASP.NET Core como Runtime e Framework Principal

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, linguagem, runtime, dotnet, csharp, aspnet-core, performance

## Contexto e Declaração do Problema

O serviço de encurtador de URLs exige altíssima vazão de rede (picos de 15.000 a 20.000 requisições por segundo) e latência P99 estrita (< 15ms) no redirecionamento. A aplicação precisa se integrar de forma eficiente com o cluster de ScyllaDB (via driver binário CQL) e clusters dedicados de Redis (via protocolo RESP), além de orquestrar concorrência assíncrona de alto volume para o buffer em memória da telemetria ([ADR-004](004-pipeline-de-analytics-com-isolamento-e-microbatching.md)).

Faz-se necessária a escolha de um runtime moderno que balanceie velocidade de desenvolvimento, segurança de tipos, ecossistema maduro e capacidade comprovada de I/O assíncrono com baixo overhead de alocação.

## Critérios de Decisão (Decision Drivers)

- **Performance e Throughput de Rede:** Capacidade de responder milhares de requisições por segundo com baixo consumo de CPU por requisição (Minimal APIs).
- **Primitivas Concorrentes Nativas:** Suporte robusto a canais assíncronos não-bloqueantes (`System.Threading.Channels`) para implementar o micro-batching da [ADR-004](004-pipeline-de-analytics-com-isolamento-e-microbatching.md).
- **Maturidade dos Drivers de Conectividade:** Existência de drivers maduros e com suporte a *connection pooling* para Cassandra/ScyllaDB (`CassandraCSharpDriver`) e Redis (`StackExchange.Redis`).
- **Observabilidade Integrada:** Suporte de primeira classe a métricas Prometheus, tracing distribuído com OpenTelemetry e estruturação de logs.
- **Suporte a Testes de Integração:** Facilidade de executar testes automatizados com instâncias reais de bancos e caches via Testcontainers.

## Opções Consideradas

- **Opção 1:** C# / .NET 9 com ASP.NET Core Minimal APIs (Opção Escolhida)
- **Opção 2:** Go (Golang) com Gin ou Fiber
- **Opção 3:** TypeScript / Node.js com Fastify
- **Opção 4:** Java 21 / Kotlin com Spring Boot 3 ou Quarkus

## Decisão Aprovada

Opção escolhida: **"Opção 1: C# / .NET 9 com ASP.NET Core Minimal APIs"**, devido à sua comprovada performance de I/O assíncrono (recorrente no topo dos benchmarks do TechEmpower), recursos avançados de baixo consumo de memória (`Span<T>`, `Memory<T>`, `ValueTask`) e facilidade de orquestração de background workers concorrentes.

### Detalhamento da Implementação:

1. **Arquitetura Minimal APIs:**
   * Utilização de Minimal APIs em substituição a MVC/Controllers pesados para reduzir overhead de pipeline, reflexão e alocações de middleware.
2. **Concorrência e Micro-Batching:**
   * Uso de `System.Threading.Channels.Channel.CreateBounded<ClickEvent>` para o buffer de analytics em memória, fornecendo comunicação thread-safe *producer-consumer* de alta vazão sem locks manuais.
3. **Conexões Gerenciadas como Singletons:**
   * A sessão do ScyllaDB (`ISession`) e o multiplexador do Redis (`IConnectionMultiplexer`) serão registrados como Singletons no container de injeção de dependência (`IServiceCollection`), mantendo pools de conexões TCP abertos e reutilizáveis durante toda a vida da aplicação.

### Consequências Positivas

- **Excelente Eficiência de Execução:** O motor Kestrel do ASP.NET Core é um dos servidores web mais rápidos do mercado para transporte HTTP.
- **Ecossistema Enterprise Maduro:** Facilidade de integração com bibliotecas consolidadas da indústria (`Testcontainers.Cassandra`, `Testcontainers.Redis`, `OpenTelemetry.Exporter.Prometheus.AspNetCore`).
- **Segurança e Produtividade:** Sistema de tipos forte do C# reduz bugs de concorrência e tipos nulos (`Nullable reference types`) em tempo de compilação.

### Consequências Negativas / Trade-offs

- **Ajustes de Garbage Collection:** Sob picos massivos de requisições, o heap do .NET deve ser gerenciado com cuidado (utilizando Server GC e evitando alocações desnecessárias de strings/objetos no caminho crítico) para impedir pausas de GC que possam impactar o P99.
- **Tamanho da Imagem de Contêiner:** A imagem Docker do runtime do .NET (~80MB a 120MB) é maior do que um binário estático puro em Go ou Rust (~15MB a 30MB).

## Prós e Contras das Opções

### Opção 1: C# / .NET 9 (ASP.NET Core) ✅ Escolhida

- ✅ **Performance de ponta:** Kestrel e Minimal APIs com baixo overhead de alocação.
- ✅ **Canais assíncronos nativos:** `System.Threading.Channels` ideal para micro-batching.
- ✅ **Drivers oficiais consolidados:** Suporte de primeira classe para Cassandra/ScyllaDB e Redis.
- ❌ **Footprint de memória:** Maior do que binários estáticos compilados em Go ou Rust.

### Opção 2: Go (Golang)

- ✅ **Binário único e baixo consumo de RAM:** Ideal para inicializações instantâneas de pods.
- ✅ **Goroutines nativas leves.**
- ❌ **Menor expressividade em modelagem de domínio complexo:** Sistema de tipos mais simples e ausência de construções avançadas presentes no C#.

### Opção 3: TypeScript / Node.js

- ✅ **Agilidade de prototipagem rápida.**
- ❌ **Event Loop Single-Threaded:** Gargalos em processamento intensivo de cifras (Feistel) ou serialização massiva de buffers sem clusters de processos.

### Opção 4: Java / Spring Boot 3

- ✅ **Ecossistema corporativo gigantesco.**
- ❌ **Consumo de memória e tempo de inicialização elevados:** Mesmo com Virtual Threads, o consumo base de heap e tempo de boot são superiores ao .NET 9 e Go.

## Links

- [ADR-001: Alocação de Faixas para Geração de Tokens Curtos](001-alocacao-de-faixas-para-geracao-de-tokens.md)
- [ADR-002: Adoção do ScyllaDB como Banco Primário](002-adocao-do-scylladb-como-banco-primario.md)
- [ADR-003: Arquitetura de Cache e Resiliência](003-arquitetura-de-cache-e-resiliencia.md)
- [ADR-004: Pipeline de Analytics com Isolamento e Micro-Batching](004-pipeline-de-analytics-com-isolamento-e-microbatching.md)
- [Especificação do Desafio](../../CHALLENGE.md)

