# ADR-004: Pipeline de Analytics com Isolamento de Instâncias e Micro-Batching em Memória

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, analytics, redis-streams, micro-batching, resiliencia, alta-disponibilidade, spof

## Contexto e Declaração do Problema

Cada redirecionamento de link gera um evento de telemetria (timestamp UTC, hash anonimizado de IP, User-Agent, Referer e geolocalização). Em horários de pico, o tráfego atinge de 15.000 a 20.000 acessos por segundo (10 bilhões de cliques/mês).

O caminho crítico de redirecionamento possui um SLA rigoroso de **P99 < 15ms** e o sistema possui o requisito mandatário de **não possuir Single Point of Failure (SPOF)**.

Dois conflitos técnicos graves emergem:
1. **Conflito de Políticas de Memória no Redis:** O cache de links exige evicção agressiva (`allkeys-lru`), enquanto streams de mensagens de analytics exigem retenção estrita (`noeviction`). Manter ambos na mesma instância física geraria risco de descarte acidental de métricas ou esgotamento de memória.
2. **Saturação de I/O de Rede e CPU:** Fazer 18.000 chamadas remotas de rede por segundo para enviar cada evento individualmente sobrecarrega a CPU do Redis e compete com as leituras de cache.

## Critérios de Decisão (Decision Drivers)

- **Eliminação de Single Point of Failure (SPOF):** Toda a infraestrutura deve possuir redundância automática (Multi-AZ com replicação primário/réplica).
- **Isolamento de Domínios de Falha:** Problemas no pipeline de analytics (ex: lentidão de workers ou acúmulo de fila) não podem afetar a latência ou a disponibilidade do redirecionamento de URLs.
- **Latência de Redirecionamento Imune à Telemetria:** O registro do clique não deve adicionar saltos de rede síncronos no caminho crítico da resposta HTTP.
- **Eficiência de Custos:** Viabilizar o tráfego de 10 bilhões de eventos mensais sem incorrer em custos proibitivos por requisição (como os US$ 8.000/mês estimados para SQS).

## Opções Consideradas

- **Opção 1:** Instância única de Redis compartilhada com ingestão síncrona individual por clique.
- **Opção 2:** Amazon SQS / SNS com workers serverless.
- **Opção 3:** Apache Kafka / Redpanda Cluster dedicado.
- **Opção 4:** Separação Física de Clusters de Redis (Multi-AZ) + Micro-Batching em Memória na Aplicação (Opção Escolhida).

## Decisão Aprovada

Opção escolhida: **"Opção 4: Separação Física de Clusters de Redis (Multi-AZ) + Micro-Batching em Memória na Aplicação"**, combinando isolamento físico de infraestrutura com buffering de altíssima performance em processo.

### Arquitetura Detalhada da Solução:

```
[ Usuário HTTP ] 
       │ (1) Request Redirect
       ▼
┌─────────────────────────────────────────────────────────────┐
│ API Service (ASP.NET Core / Go)                             │
│  ├── 1. Consulta `redis-cache` (Hit < 2ms)                  │
│  ├── 2. Responde HTTP 307 imediatamente                     │
│  └── 3. Despacha ClickEvent no Channel em Memória (0ms rede)│
│                                                             │
│  Background Task (Buffer em Processo)                       │
│  └── Acumula cliques: Limite 100ms OU 500 itens             │
│      └── Dispara Pipeline em Lote `XADD` ────────────────┐  │
└──────────────────────────────────────────────────────────┼──┘
                                                           │ (Batch)
                                                           ▼
                                           ┌───────────────────────────────┐
                                           │ Cluster `redis-stream`        │
                                           │ (Multi-AZ / NoEviction)       │
                                           └───────────────┬───────────────┘
                                                           │ (Consumer Group)
                                                           ▼
                                           ┌───────────────────────────────┐
                                           │ Analytics Consumer Worker     │
                                           │ └── Grava agregações em lote  │
                                           └───────────────┬───────────────┘
                                                           ▼
                                           ┌───────────────────────────────┐
                                           │ ScyllaDB (Métricas Agregadas) │
                                           └───────────────────────────────┘
```

1. **Separação de Infraestrutura (Zero SPOF):**
   * **Cluster `redis-cache`:** Configurado em Multi-AZ com failover automático, política `allkeys-lru`, exclusivo para cache de URLs e alocação de faixas de sequências ([ADR-001](001-alocacao-de-faixas-para-geracao-de-tokens.md)).
   * **Cluster `redis-stream`:** Configurado em Multi-AZ com failover automático, política `noeviction` e teto de tamanho no stream (`MAXLEN ~ 1000000`). Dedicado exclusivamente à mensageria de analytics.
2. **Buffer Local de Alta Performance (Micro-Batching):**
   * Na requisição de redirecionamento, o evento é gravado em um canal não-bloqueante em RAM (`System.Threading.Channels` em .NET ou *Buffered Channel* em Go).
   * O cliente recebe o redirecionamento HTTP **sem nenhum salto de rede adicional para analytics** (custo de rede = 0ms).
   * Uma background task local monitora o canal e faz flush para o `redis-stream` em lote com base em duas condições: **a cada 100 milissegundos** OU **quando atingir 500 eventos** (o que acontecer primeiro).
   * Isso reduz 18.000 requisições individuais de rede por segundo para apenas **~36 a 50 operações de lote por segundo** no `redis-stream`.
3. **Consumo e Persistência no ScyllaDB:**
   * Workers independentes consom do Redis Stream via `XREADGROUP` em blocos, realizam agregação por janela temporal (hora/dia) e persistem no ScyllaDB.

### Consequências Positivas

- **Zero SPOF e Isolamento Total:** Uma sobrecarga ou manutenção no cluster de analytics nunca derrubará o cluster de redirecionamento/cache.
- **Caminho Crítico Ultra-Rápido:** A latência percebida pelo usuário no redirect é pura leitura de cache em RAM.
- **Redução de Carga de 99% na Rede:** O micro-batching em processo colapsa milhares de chamadas em poucas dezenas de transações em lote.
- **Economia Brutal de Custos:** Ambos os clusters Redis somam menos de US$ 200/mês, eliminando os US$ 8.000/mês de filas pagas por requisição (SQS).

### Consequências Negativas / Trade-offs

- **Tolerância a Perda Mínima em Crash Repentino:** Se um pod da API for encerrado abruptamente via `kill -9` ou falha de hardware sem execução de *Graceful Shutdown*, os eventos retidos no buffer de até 100ms daquela instância específica podem ser perdidos. *(Trade-off plenamente aceitável para métricas estatísticas em relação ao ganho de performance e custo).*
- **Gerenciamento de Dois Clusters:** Requer o provisionamento e monitoramento de dois endpoints distintos de Redis no ambiente de infraestrutura.

## Prós e Contras das Opções

### Opção 4: Dois Clusters Redis Multi-AZ + Micro-Batching em Memória ✅ Escolhida

- ✅ **Elimina Single Point of Failure (SPOF).**
- ✅ **Isolamento de políticas:** Cache pode usar LRU livremente; Stream usa retenção garantida.
- ✅ **Latência zero de rede para analytics no caminho do redirect.**
- ❌ **Custo ligeiramente maior que 1 único Redis** (~$190/mês vs ~$95/mês, mas ordens de grandeza menor que SQS).

### Opção 1: Redis Único Compartilhado

- ✅ **Menor custo de infraestrutura.**
- ❌ **Conflito destrutivo de memória:** `allkeys-lru` pode apagar eventos do stream antes do consumo.
- ❌ **Gargalo de CPU:** Concorrência entre 18k leituras de cache e 18k escritas de stream no mesmo core.

### Opção 2: Amazon SQS / SNS

- ✅ **Sem servidores para gerenciar.**
- ❌ **Custo inviável:** ~US$ 8.000/mês para 10 bilhões de mensagens.
- ❌ **Latência excessiva:** Chamadas HTTPS adicionam 15 a 25ms de latência, violando o SLA de P99.

### Opção 3: Apache Kafka / Redpanda

- ✅ **Capacidade analítica massiva.**
- ❌ **Alta complexidade operacional:** Exige gestão de partições, zookeeper/kraft, balanceamento de partições e maior consumo de infraestrutura base.

## Links

- [ADR-001: Alocação de Faixas para Geração de Tokens Curtos](001-alocacao-de-faixas-para-geracao-de-tokens.md)
- [ADR-002: Adoção do ScyllaDB como Banco de Dados Primário](002-adocao-do-scylladb-como-banco-primario.md)
- [ADR-003: Arquitetura de Cache e Resiliência](003-arquitetura-de-cache-e-resiliencia.md)
- [Especificação do Desafio](../../CHALLENGE.md)

