# ADR-006: Padronização Exclusiva de Redirecionamento HTTP 307 com Headers Anti-Cache

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, http, redirecionamento, analytics, seguranca, rfc7231, yagni

## Contexto e Declaração do Problema

O protocolo HTTP oferece múltiplos códigos de status para redirecionamento (`301`, `302`, `307`, `308`), cada um com regras distintas de armazenamento em cache pelos navegadores e tratamento do método HTTP original da requisição.

O sistema possui duas necessidades fundamentais:
1. **Acurácia Absoluta na Telemetria:** Garantir que 100% dos cliques sejam registrados no pipeline de analytics ([ADR-004](004-pipeline-de-analytics-com-isolamento-e-microbatching.md)).
2. **Revogabilidade e Expiração Imediatas:** Se um link expirar (TTL) ou for identificado como phishing/malware, o acesso deve ser bloqueado instantaneamente para qualquer usuário.

Se o navegador do usuário armazenar a resposta de redirecionamento em cache de disco local, requisições futuras do mesmo usuário nunca mais baterão nos servidores do encurtador, quebrando a contagem de métricas e tornando impossível impedir o acesso a links maliciosos já cacheados no cliente.

## Critérios de Decisão (Decision Drivers)

- **Garantia de Consulta Contínua:** O navegador deve obrigatoriamente consultar o servidor encurtador em cada clique.
- **Segurança e Anti-Abuso:** Capacidade de banir, pausar ou expirar um link em tempo real com efeito global imediato.
- **Preservação de Métodos HTTP (RFC 7231):** Redirecionar requisições sem reescrever métodos (`POST`, `PUT`, `DELETE` para `GET`).
- **Simplicidade de Domínio (YAGNI):** Evitar complexidade desnecessária de suportar múltiplos tipos de redirecionamento configuráveis por link.

## Opções Consideradas

- **Opção 1:** HTTP 301 Moved Permanently (ou opção mista 301/307 configurável pelo usuário).
- **Opção 2:** HTTP 302 Found clássico.
- **Opção 3:** HTTP 307 Temporary Redirect exclusivo acompanhado de Headers de Controle Anti-Cache (Opção Escolhida).

## Decisão Aprovada

Opção escolhida: **"Opção 3: HTTP 307 Temporary Redirect exclusivo com Headers Anti-Cache"**, eliminando o suporte a 301/redirecionamento permanente no sistema.

### Detalhamento da Resposta HTTP no ASP.NET Core:

Todo redirecionamento bem-sucedido responderá estritamente com:

* **Status Code:** `307 Temporary Redirect`
* **Headers de Resposta:**
  ```http
  Location: https://url-de-destino-original.com
  Cache-Control: private, max-age=0, no-cache, no-store, must-revalidate
  Pragma: no-cache
  Expires: 0
  ```

### Consequências Positivas

- **Telemetria 100% Confiável:** Cada acesso atinge o encurtador e é contabilizado no pipeline de métricas.
- **Imunidade contra Links Maliciosos "Presos em Cache":** A exclusão de um link no ScyllaDB ou a invalidação no Redis surte efeito imediato em todos os navegadores globalmente.
- **Simplicidade de Modelo de Dados:** Não há necessidade de armazenar `redirect_type` ou `status_code` na tabela de URLs, mantendo o payload leve e eliminando código condicional na API.
- **Conformidade Estrita com RFC 7231:** Clientes programáticos e APIs que chamem links encurtados mantêm seus métodos HTTP originais.

### Consequências Negativas / Trade-offs

- **Toda Requisição Bate na Infraestrutura:** Não há desoneração de tráfego por cache de disco no navegador do cliente *(trade-off plenamente mitigado pelo Redis Cache-Aside da [ADR-003](003-arquitetura-de-cache-e-resiliencia.md), que responde leituras quentes em RAM em < 2ms).*

## Prós e Contras das Opções

### Opção 3: HTTP 307 com Headers Anti-Cache ✅ Escolhida

- ✅ **Métricas precisas:** Elimina subnotificação de cliques.
- ✅ **Controle total de ciclo de vida:** TTL e revogação funcionam instantaneamente.
- ✅ **Design enxuto (YAGNI):** Regra única para todo o sistema.
- ❌ **Tráfego constante:** Cada clique gera requisição de rede ao servidor.

### Opção 1: Suporte a HTTP 301 (Permanente)

- ✅ **Economiza largura de banda do servidor.**
- ❌ **Destrói as métricas de analytics:** Cliques subsequentes do mesmo cliente nunca chegam ao servidor.
- ❌ **Risco de segurança:** Links maliciosos banidos continuam abrindo no cache do navegador da vítima.
- ❌ **Complexidade inútil:** Gera chamados de suporte de usuários confusos com métricas discrepantes.

### Opção 2: HTTP 302 Found

- ✅ **Evita cache agressivo por padrão.**
- ❌ **Comportamento ambíguo:** Clientes HTTP antigos podem transformar requisições `POST` em `GET` durante o redirecionamento.

## Links

- [ADR-003: Arquitetura de Cache e Resiliência](003-arquitetura-de-cache-e-resiliencia.md)
- [ADR-004: Pipeline de Analytics com Isolamento e Micro-Batching](004-pipeline-de-analytics-com-isolamento-e-microbatching.md)
- [ADR-005: Adoção do ASP.NET Core como Runtime Principal](005-adocao-do-aspnet-core-como-framework-principal.md)
- [Especificação do Desafio](../../CHALLENGE.md)

