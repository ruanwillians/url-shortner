# ADR-002: Adoção do ScyllaDB como Banco de Dados Primário de Persistência

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, banco-de-dados, nosql, scylladb, cql, persistencia, performance, ttl

## Contexto e Declaração do Problema

O sistema de encurtador de URLs foi projetado para armazenar até 6 bilhões de registros ao longo de 5 anos (~3,9 TB de dados brutos) com um padrão de acesso primordialmente de leitura direta pontual por chave (`short_code` $\rightarrow$ `original_url`). O tráfego de leitura atinge picos de 15.000 a 20.000 QPS e as escritas variam entre 38 e 300 requisições por segundo.

Além do volume de armazenamento, URLs podem possuir data de expiração configurável (TTL). Em bancos relacionais tradicionais, a deleção periódica de centenas de milhões de registros expirados gera inchaço de tabela (*table bloat* / *dead tuples*), exigindo rotinas pesadas de `VACUUM` que competem por I/O com as operações críticas de leitura.

## Critérios de Decisão (Decision Drivers)

- **Suporte nativo a TTL por registro:** O banco deve expirar dados automaticamente no disco sem necessidade de workers ou cron jobs de limpeza contínua.
- **Latência ultra-baixa e previsível (P99 estável):** Eliminar picos de latência decorrentes de pausas de Garbage Collection ou contenção de locks de páginas.
- **Escalabilidade linear para múltiplos Terabytes:** Capacidade de particionar e distribuir 3,9 TB ou mais de dados entre nós sem sharding complexo na camada de aplicação.
- **Aderência ao modelo de acesso Key-Value:** Otimização para leituras pontuais por chave de partição única (`short_code`).

## Opções Consideradas

- **Opção 1:** PostgreSQL com Particionamento Declarativo por Hash
- **Opção 2:** ScyllaDB Distribuído (Wide-Column NoSQL via CQL)
- **Opção 3:** Amazon DynamoDB (Gerenciado Serverless)
- **Opção 4:** MongoDB com Sharding

## Decisão Aprovada

Opção escolhida: **"Opção 2: ScyllaDB Distribuído"**, porque atende nativamente ao modelo de acesso por chave única em larga escala, oferece TTL nativo por linha e garante latência P99 sub-milissegundo com arquitetura C++ *thread-per-core* sem pausas de Garbage Collector.

### Modelagem da Tabela Principal no ScyllaDB (`urls`)

```sql
CREATE KEYSPACE IF NOT EXISTS url_shortener 
WITH replication = {
    'class': 'NetworkTopologyStrategy', 
    'replication_factor': 3
};

CREATE TABLE IF NOT EXISTS url_shortener.urls (
    short_code text,
    original_url text,
    created_at timestamp,
    user_id uuid,
    is_active boolean,
    PRIMARY KEY (short_code)
);
```

Para gravações com expiração temporária, o sistema utilizará a cláusula nativa de TTL:
```sql
INSERT INTO url_shortener.urls (short_code, original_url, created_at, user_id, is_active)
VALUES (?, ?, ?, ?, ?)
USING TTL ?;
```

### Integração com a ADR-001 (Alocação de Faixas de IDs)
Como o ScyllaDB não possui sequences relacionais (`CREATE SEQUENCE`), a alocação de blocos numéricos estabelecida na [ADR-001](001-alocacao-de-faixas-para-geracao-de-tokens.md) será coordenada atomicamente pela camada de **Redis** via comando `INCRBY url_sequence_block 100000`.

### Consequências Positivas

- **Zero manutenção de links expirados:** A limpeza de dados com TTL ocorre silenciosamente durante a compactação nativa das SSTables no ScyllaDB, sem impacto de performance e sem gerar dead tuples.
- **Performance extrema e previsível:** A arquitetura em C++ (framework Seastar) do ScyllaDB elimina pausas de GC da JVM do Cassandra clássico, mantendo leituras frias em ~1ms.
- **Crescimento horizontal indolor:** Aumento de volume de dados ou tráfego é resolvido adicionando nós ao anel do cluster sem interrupção de serviço.

### Consequências Negativas / Trade-offs

- **Custo base de infraestrutura:** Requer um cluster mínimo de 3 nós para quórum e redundância (`replication_factor: 3`), tornando o custo de entrada na nuvem mais alto que uma instância única inicial de PostgreSQL.
- **Sem suporte a transações ACID relacionais complexas:** Aliases personalizados (*vanity URLs*) exigem uso de Lightweight Transactions (LWT) com `IF NOT EXISTS` para evitar sobreposição simultânea de chaves customizadas.
- **Curva de aprendizado:** Exige uso do protocolo CQL e drivers específicos de Cassandra/ScyllaDB em vez do padrão relacional tradicional.

## Prós e Contras das Opções

### Opção 2: ScyllaDB Distribuído ✅ Escolhida

- ✅ **TTL nativo por linha:** Remove completamente a necessidade de workers de expiração.
- ✅ **Throughput massivo por nó:** Escrita e leitura direta em disco NVMe com arquitetura *shared-nothing*.
- ✅ **Compatibilidade:** Suporta o ecossistema CQL (drivers para .NET, Go, Java, Rust).
- ❌ **Custo de entrada:** Cluster mínimo de 3 nós recomendados em produção.

### Opção 1: PostgreSQL Particionado

- ✅ **Econômico inicialmente:** Permite começar com uma instância pequena no RDS.
- ✅ **ACID completo:** Transações e sequences nativas.
- ❌ **Inchaço de tabela com deletes:** Limpar centenas de milhões de URLs expiradas sobrecarrega o `VACUUM`.
- ❌ **Degradação de índice:** Índices B-Tree para 6 bilhões de registros exigem muita memória RAM para evitar leituras de disco lentas.

### Opção 3: Amazon DynamoDB

- ✅ **Totalmente gerenciado / Serverless:** Zero manutenção de servidores.
- ✅ **TTL nativo integrado.**
- ❌ **Custo proibitivo em alta escala:** Cobrança por WCU (Write Capacity Units) e RCU (Read Capacity Units) com 10 bilhões de acessos mensais gera custos astronômicos se o tráfego não estiver 100% em cache.
- ❌ **Vendor lock-in:** Amarra a arquitetura exclusivamente à AWS, dificultando testes locais com alta fidelidade.

### Opção 4: MongoDB com Sharding

- ✅ **Índices TTL integrados.**
- ❌ **Sobrecarga de armazenamento:** O formato BSON adiciona overhead em bytes por documento, encarecendo o armazenamento de 6 bilhões de registros.
- ❌ **Performance inferior a wide-column:** Menor eficiência de I/O em comparação direta com a engine LSM do ScyllaDB para chave-valor puro.

## Links

- [ADR-001: Alocação de Faixas para Geração de Tokens Curtos](001-alocacao-de-faixas-para-geracao-de-tokens.md)
- [Especificação do Desafio](../../CHALLENGE.md)
