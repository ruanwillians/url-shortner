# ADR-003: Arquitetura de Cache, Mitigação de Cache Stampede e Penetration

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, cache, redis, performance, resiliencia, singleflight

## Contexto e Declaração do Problema

Com um pico de leitura de 15.000 a 20.000 QPS e um SLA estrito de latência em P99 < 15ms, a camada de banco de dados (ScyllaDB) não deve ser consultada para todo redirecionamento. O tráfego de encurtadores de links é tipicamente assimétrico e segue a Regra de Pareto (20% das URLs concentram mais de 80% dos acessos).

Três problemas críticos de resiliência e concorrência precisam ser solucionados:
1. **Cache Stampede (Thundering Herd):** Quando o cache de um link de altíssimo tráfego expira, centenas ou milhares de requisições simultâneas sofrem cache miss no mesmo milissegundo e disparam consultas idênticas ao banco de dados ao mesmo tempo.
2. **Cache Penetration:** Ataques ou bots requisitando chaves que não existem forçam cache misses contínuos, bypassando o cache e sobrecarregando o banco primário com leituras inúteis.
3. **Limitação de Memória RAM:** É financeiramente inviável manter 6 bilhões de URLs em memória RAM do Redis.

## Critérios de Decisão (Decision Drivers)

- **Latência P99 mínima (< 5ms no cache):** O caminho de redirecionamento de links quentes deve responder quase que instantaneamente.
- **Blindagem do ScyllaDB:** Garantir que o banco de dados receba no máximo 1 consulta por chave em caso de cache miss concorrente.
- **Eficiência de custos de memória:** O cache deve gerenciar seu próprio ciclo de vida descartando links frios automaticamente.
- **Proteção contra varreduras maliciosas:** Impedir que consultas a códigos inexistentes atinjam o ScyllaDB repetidamente.

## Opções Consideradas

- **Opção 1:** Redis Cache-Aside básico sem mecanismos de contenção de concorrência.
- **Opção 2:** Redis Cache-Aside com Locks Distribuídos (Mutex via Redis/Redlock).
- **Opção 3:** Redis Cache-Aside com Singleflight em Processo + Negative Caching (Cache de Nulos) + TTL com Jitter + Política `allkeys-lru` (Opção Escolhida).

## Decisão Aprovada

Opção escolhida: **"Opção 3: Redis Cache-Aside com Singleflight + Negative Caching + TTL com Jitter + allkeys-lru"**, implementando uma defesa em camadas sem o overhead e a complexidade de locks distribuídos de rede.

### Detalhamento da Arquitetura de Cache:

1. **Padrão de Acesso (Cache-Aside):**
   * A aplicação consulta o Redis (`GET url:{short_code}`).
   * Se hit: redireciona imediatamente.
   * Se miss: recorre ao mecanismo de Singleflight antes de consultar o ScyllaDB.
2. **Mitigação de Stampede (Singleflight em Processo):**
   * A aplicação utiliza um mecanismo de deduplicação de chamadas concorrentes em memória (padrão *Singleflight*).
   * Se 1.000 requisições simultâneas chegarem para o mesmo `short_code` em miss, apenas 1 thread faz a consulta ao ScyllaDB; as outras 999 aguardam o resultado da mesma promessa/thread em memória e compartilham o retorno.
3. **Prevenção de Expirações Sincronizadas (TTL com Jitter):**
   * O TTL base das chaves no Redis é configurado para 24 horas, adicionando-se uma variação pseudoaleatória (Jitter) de $\pm 10\%$ (entre 21h36 e 26h24). Isso evita que lotes massivos de links criados no mesmo horário expirem simultaneamente.
4. **Mitigação de Penetration (Negative Caching):**
   * Se uma chave for consultada e não existir no ScyllaDB, grava-se no Redis um valor sentinela vazio (`__NULL__`) com TTL curto de **60 segundos**. Requisições repetidas para a mesma chave inexistente recebem 404 direto do cache sem tocar no banco.
5. **Política de Evicção do Redis:**
   * Configuração de `maxmemory-policy allkeys-lru`. Quando a memória atinge o limite alocado, o Redis descarta as chaves menos recentemente acessadas, mantendo em RAM apenas o conjunto quente.

### Consequências Positivas

- **Proteção total do ScyllaDB:** Singleflight extingue o efeito Thundering Herd dentro de cada réplica da API.
- **Imunidade a bots de varredura:** Negative caching neutraliza ataques de enumeração aleatória.
- **Controle rígido de custos:** A política `allkeys-lru` permite dimensionar o Redis com base no orçamento de RAM (ex: nós de 13 GB a 26 GB), independentemente do tamanho total do banco de dados (3,9 TB).

### Consequências Negativas / Trade-offs

- **Singleflight é local por instância:** Em clusters com muitas réplicas da API, cada réplica fará 1 query ao banco no momento do miss (ex: com 10 réplicas, serão no máximo 10 queries em vez de 1.000, o que o ScyllaDB absorve sem nenhum esforço).
- **Invalidação de Cache:** Se um link for desativado ou alterado antes da expiração, a aplicação deve disparar um comando `DEL url:{short_code}` no Redis.

## Prós e Contras das Opções

### Opção 3: Cache-Aside com Singleflight + Negative Caching + Jitter ✅ Escolhida

- ✅ **Alta performance:** Zero latência adicional de locks remotos distribuídos.
- ✅ **Resiliente a falhas:** Desacoplado; se o banco primário oscilar, o cache continua respondendo o tráfego quente.
- ❌ **Complexidade no código da aplicação:** Requer a implementação correta da abstração de Singleflight.

### Opção 1: Cache-Aside Básico

- ✅ **Implementação trivial.**
- ❌ **Extremamente vulnerável a picos virais:** Qualquer link popular expirando derruba ou satura o banco de dados.
- ❌ **Vulnerável a bots de scraping.**

### Opção 2: Mutex Distribuído com Redlock

- ✅ **Garante 1 única chamada global entre todos os pods.**
- ❌ **Latência adicional:** Requer chamadas de rede para adquirir e liberar locks no Redis para cada miss.
- ❌ **Risco de Deadlocks:** Se uma instância travar enquanto segura o lock, as demais sofrem timeout.

## Links

- [ADR-002: Adoção do ScyllaDB como Banco de Dados Primário](002-adocao-do-scylladb-como-banco-primario.md)
- [Especificação do Desafio](../../CHALLENGE.md)

