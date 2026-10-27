# Senior Engineering Challenge: High-Throughput & Fault-Tolerant URL Shortener

## 1. Visão Geral do Desafio

Construir um **serviço de encurtamento de URL resiliente, observável e de baixa latência**, desenhado para operar em escala de internet com uma razão de leitura/escrita acentuada (Read-Heavy).

O objetivo deste desafio não é apenas implementar um CRUD simples, mas resolver dilemas reais de sistemas distribuídos:
* Geração determinística ou pré-alocada de chaves sem colisão e sem single point of failure (SPOF).
* Latência de redirecionamento em P99 na casa de milissegundos (< 15ms).
* Telemetria e analítica em tempo real sem degradar o caminho crítico de redirecionamento.
* Proteção contra abuso, scraping malicioso e phishing.
* Resiliência a *cache stampede* (thundering herd) e falhas parciais de infraestrutura.

---

## 2. Dimensionamento & Capacidade (The Back-of-the-Envelope Math)

O sistema deve ser projetado considerando o seguinte perfil de tráfego:

* **Novas URLs encurtadas por mês:** 100 milhões (~38 escritas/segundo em média, com picos de 300 writes/sec).
* **Taxa Leitura/Escrita:** 100:1 (para cada escrita, ocorrem 100 acessos).
* **Redirecionamentos por mês:** 10 bilhões (~3.850 leituras/segundo em média, picos de 15.000 a 20.000 QPS).
* **Tamanho médio do payload:**
  * URL Original: ~500 bytes.
  * Chave curta (alias): 7 a 8 caracteres Base62.
  * Metadados (timestamp, status, expiration, owner_id): ~150 bytes.
  * Total por registro: ~650 bytes.
* **Retenção de dados:** 5 anos por padrão (a menos que possua TTL explícito).
* **Armazenamento estimado em 5 anos:**
  $$100 \times 10^6 \times 12 \times 5 = 6 \text{ bilhões de registros}$$
  $$6 \times 10^9 \times 650 \text{ bytes} \approx 3.9 \text{ TB}$$
* **Throughput de Rede estimado (Pico de Leitura):**
  $$20.000 \text{ req/s} \times 1 \text{ KB (HTTP headers + payload)} \approx 20 \text{ MB/s (160 Mbps)}$$

---

## 3. Requisitos Funcionais (RF)

### 3.1. Encurtamento de URL
* **RF01 - Geração de Token Único:** O sistema deve gerar um identificador curto de 7 ou 8 caracteres alfanuméricos (`[a-zA-Z0-9]` - Base62).
  * Exemplo: `https://sh.rt/xK9z2Pq`.
* **RF02 - Alias Customizado (Vanity URL):** O usuário pode solicitar um alias específico (ex: `https://sh.rt/blackfriday`).
  * O sistema deve validar conflitos, tamanho mínimo/máximo e palavras reservadas (`/api`, `/metrics`, `/health`, etc.).
* **RF03 - Expiração (TTL):** Opcionalmente, uma URL pode ter uma data/hora de expiração. Após expirada, deve retornar `404 Not Found` ou `410 Gone`.
* **RF04 - Idempotência na Criação:** Requisições idênticas para a mesma URL original vindas do mesmo cliente/tenant dentro de uma janela de tempo devem retornar o mesmo token, a menos que custom alias ou flags forcem novos links.

### 3.2. Redirecionamento
* **RF05 - Redirecionamento HTTP:**
  * Suporte a `302 Found` ou `307 Temporary Redirect` (para garantir que requisições passem pelos servidores e gerem métricas analíticas).
  * Suporte a `301 Moved Permanently` configurável para casos onde a economia de banda e cache no browser do cliente forem prioridade máxima sobre telemetria em tempo real.
* **RF06 - Validação e Tratamento de Erro:**
  * Se o token não existir: retornar `404 Not Found`.
  * Se estiver expirado: retornar `410 Gone`.

### 3.3. Analytics & Telemetria
* **RF07 - Ingestão Assíncrona de Métricas:** O caminho crítico de redirecionamento **nunca** deve aguardar o banco de dados de analytics.
  * Coletar: Timestamp UTC, IP do cliente (anonimizado/truncado por LGPD/GDPR), User-Agent, Referer, País/Região (via GeoIP).
* **RF08 - Endpoint de Estatísticas:**
  * Fornecer consulta agregada por link: Total de cliques, cliques por período (hora, dia), top países, top referers, browsers/sistemas operacionais.

---

## 4. Requisitos Não Funcionais (RNF)

| Dimensão | Requisito / Alvo |
| :--- | :--- |
| **Disponibilidade** | **99.99%** (tolerância máxima de ~4.3 minutos de downtime por mês). |
| **Latência de Redirecionamento** | **P95 < 5ms**, **P99 < 15ms** em nós locais com cache. |
| **Latência de Criação** | **P99 < 100ms**. |
| **Consistência** | *Read-After-Write* consistente para a criação do link (o link criado deve poder ser acessado imediatamente); *Eventual Consistency* aceitável para analytics (< 30s de delay). |
| **Tolerância a Falhas** | A indisponibilidade do broker de analytics não deve derrubar o redirecionamento. A queda temporária da réplica primária de escrita não deve parar as leituras (fallback em cache/réplicas). |
| **Escalabilidade Horizontal** | O serviço de redirecionamento deve ser *stateless* e autoescalável com base em CPU e QPS. |

---

## 5. Requisitos de Segurança & Integridade

* **SEC01 - Prevenção contra Abuso & Scraping:** Rate limiting por IP e por API Key (ex: Leaky Bucket ou Token Bucket com Redis).
* **SEC02 - Anti-Phishing & Safe Browsing:** Validação da URL original contra listas maliciosas (Google Safe Browsing API ou listas de blocklist locais) antes de persistir o link.
* **SEC03 - Proteção contra Loop de Redirecionamento:** Rejeitar URLs que apontem recursivamente para o próprio domínio encurtador.
* **SEC04 - Sanitização e Validação:** Bloqueio de esquemas não permitidos (apenas `http://` e `https://`). Prevenção contra XSS em campos de metadados e SQL Injection.

---

## 6. Trade-offs Arquiteturais Críticos para Você Resolver

Como Engenheiro(a) Sênior, você deverá justificar suas escolhas técnicas:

1. **Estratégia de Geração de Chaves:**
   * *Abordagem A:* Hashing (MD5 / Murmur3 / SHA-256) truncado em Base62 + Detecção de Colisão. (Desafio: queries extras em colisão).
   * *Abordagem B:* Key Generation Service (KGS) pré-gerando chaves únicas em background e gravando em buffer local / fila. (Desafio: gerenciar concorrência e chaves descartadas).
   * *Abordagem C:* Distributed ID Generator (Snowflake / Counter em blocos distribuídos tipo Flickr/Instagram ticket server) convertido para Base62.
2. **Estratégia de Caching:**
   * Como evitar **Cache Penetration** (requisições em massa para chaves inexistentes)? *(Dica: Bloom Filter ou caching de null values com TTL curto)*.
   * Como mitigar **Cache Stampede / Thundering Herd** em links de alto tráfego quando o cache expirar? *(Dica: Probabilistic early expiration - XFetch, ou Mutex Locking)*.
3. **Estratégia de Limpeza de Links Expirados:**
   * Evicção preguiçosa (*Lazy Deletion* na leitura) vs. *TTL no banco* vs. *Job de background (Worker)*.
4. **Desacoplamento do Pipeline de Métricas:**
   * Buffer in-memory com flush em lote vs. Message Broker (Kafka / RabbitMQ / Redis Streams / SQS).

---

## 7. Critérios de Avaliação e Entregáveis

Para considerar o desafio concluído em nível Sênior, o projeto deve entregar:

1. **RFC / Architecture Decision Record (ADR):**
   * Justificativa da escolha da linguagem (Go, Rust, Node, Java, C#, etc.), banco de dados (PostgreSQL, ScyllaDB, DynamoDB, etc.), cache (Redis, Dragonfly) e mensageria.
2. **Código de Produção:**
   * Clean Architecture / Hexagonal / DDD (separação entre domínio, aplicação e adaptadores).
   * Sem *magic numbers* ou configurações hardcoded (Twelve-Factor App).
   * Concorrência e *graceful shutdown* implementados corretamente.
3. **Observabilidade Pronta:**
   * Endpoint de `/health` e `/metrics` (Prometheus).
   * Structured Logging com Correlation ID / Request ID propagado.
   * OpenTelemetry Tracing configurado.
4. **Testes Automatizados:**
   * Testes Unitários de domínio.
   * Testes de Integração com contêineres reais (via Testcontainers ou Docker Compose).
   * **Script de Teste de Carga (k6 / Locust):** Provar que o sistema sustenta pelo menos 5.000 QPS de leitura com P99 < 15ms no ambiente de teste.

