# ADR-001: Alocação de Faixas para Geração de Tokens Curtos

- **Data**: 2026-10-26
- **Status**: Aceito
- **Decisores**: Time de Engenharia
- **Tags**: arquitetura, performance, tokens, algoritmo, distribuído

## Contexto e Declaração do Problema

O encurtador de URLs foi dimensionado para absorver 100 milhões de novas escritas por mês (com picos de ~300 requisições por segundo) e reter até 6 bilhões de registros ao longo de 5 anos.
O sistema deve gerar identificadores alfanuméricos compactos de 7 caracteres em Base62 (`[0-9a-zA-Z]`), cujo espaço amostral é de aproximadamente 3,52 trilhões de combinações ($62^7 \approx 3{,}52 \times 10^{12}$).

O desafio central é emitir identificadores únicos com latência de P99 em microssegundos, sem exigir roundtrips repetidos ao banco de dados para checagem de colisão, sem criar pontos únicos de gargalo operacional e sem permitir que atacantes adivinhem links por enumeração sequencial simples (`/1`, `/2`, `/3`).

## Critérios de Decisão (Decision Drivers)

- **Garantia matemática de não-colisão:** Eliminar consultas extras de retry ou checagem de colisão no caminho crítico de escrita.
- **Latência de emissão ultra-baixa:** A geração da chave deve ocorrer em $O(1)$ na memória local da aplicação (< 1ms).
- **Simplicidade operacional:** Evitar a manutenção de microsserviços dedicados ou pipelines complexos de infraestrutura (como workers KGS ou lambdas com filas de pooling).
- **Proteção contra enumeração (Scraping):** Impedir que o token gerado seja sequencial e previsível na URL pública.
- **Tamanho compacto:** Manter o tamanho estritamente em 7 caracteres (ou até 8).

## Opções Consideradas

- **Opção 1:** Hashing Criptográfico com Truncamento (MD5 / Murmur3 / SHA-256) e Resolução de Colisão
- **Opção 2:** Key Generation Service (KGS) Dedicado ou Serverless em Lote
- **Opção 3:** Alocação de Faixas de Sequência (Range Allocation) + Permutação Reversível de Feistel + Base62
- **Opção 4:** Geradores Distribuídos de 64 bits baseados em Tempo (Snowflake / ULID)

## Decisão Aprovada

Opção escolhida: **"Opção 3: Alocação de Faixas de Sequência (Range Allocation) + Permutação Reversível de Feistel + Base62"**, porque atende simultaneamente aos requisitos de zero colisão matemática, latência de emissão em memória e simplicidade operacional sem dependência de serviços externos contínuos.

### Como Funcionará na Prática:
1. **Coordenação Central de Faixas:** Um registrador central (tabela de sequências do banco de dados relacional ou comando atômico no Redis) distribui faixas de inteiros para as instâncias da API (ex: Instância A reserva `1` a `100.000`; Instância B reserva `100.001` a `200.000`).
2. **Incremento Atômico Local:** Cada réplica incrementa um contador atômico em memória local (`AtomicLong` / `atomic.AddUint64`), emitindo um novo número inteiro em microssegundos.
3. **Ofuscação via Cifra de Feistel:** O número sequencial passa por uma permutação pseudoaleatória reversível (Cifra de Feistel em 43 bits). A permutação mapeia bijetivamente cada número inteiro em outro número não-sequencial e aparentemente aleatório, sem risco de colisão.
4. **Codificação Base62:** O número resultante é convertido para a representação de 7 caracteres Base62.

### Consequências Positivas

- **Zero consultas de colisão:** Nenhuma escrita precisa fazer `SELECT` para verificar se a chave gerada já existe.
- **Emissão ultra-rápida:** 99.999% das emissões ocorrem em RAM na própria instância. Uma requisição de rede para buscar nova faixa acontece apenas 1 vez a cada 100.000 URLs emitidas.
- **Imprevisibilidade total:** O uso de Feistel impede que bots façam scraping varrendo URLs sequenciais.
- **Zero infraestrutura dedicada:** Não exige serviços KGS independentes, workers 24/7 ou custos recorrentes de funções serverless por invocação.

### Consequências Negativas / Trade-offs

- **Descarte de IDs em Crash/Restart:** Se uma instância reiniciar antes de esgotar sua faixa de 100.000 IDs, os números residuais daquela faixa são perdidos. *(Mitigação: frente a 3,52 trilhões de combinações, perder centenas de milhares de IDs é matematicamente irrelevante).*
- **Tratamento especial para Vanity URLs (Aliases Customizados):** Aliases personalizados solicitados explicitamente por usuários (ex: `/blackfriday`) não utilizam o gerador de faixas e ainda exigem validação de unicidade atômica via `UNIQUE INDEX` no banco de dados.

## Prós e Contras das Opções

### Opção 3: Alocação de Faixas + Feistel + Base62 ✅ Escolhida

- ✅ **Garantia matemática de não-colisão:** Inerente à exclusividade das faixas alocadas.
- ✅ **Performance máxima:** Emissão $O(1)$ sem saltos de rede na imensa maioria das requisições.
- ✅ **Arquitetura enxuta:** Funciona dentro do ciclo de vida da própria aplicação.
- ❌ **Lacunas na sequência:** Podem ocorrer saltos na numeração interna caso instâncias sofram restart.

### Opção 1: Hashing com Truncamento (MD5 / Murmur3)

- ✅ **Deduplicação automática:** Mesma URL original sem alias pode gerar o mesmo token.
- ❌ **Colisões inevitáveis:** Conforme o banco cresce, o paradoxo do aniversário força queries de retry repetidas (`SELECT` + novo `INSERT`).
- ❌ **Degradação de P99 na escrita:** Picos de colisão aumentam drasticamente a latência percebida pelo usuário.

### Opção 2: Key Generation Service (KGS)

- ✅ **Chaves pré-geradas:** Elimina cálculos na API de ingestão.
- ❌ **Sobrecarga operacional:** Exige manter workers contínuos ou arquitetura serverless (EventBridge + Lambda + SQS) monitorando buffers de chaves.
- ❌ **Custos e locks de consumo:** Consumir chaves individualmente de um pool remoto via rede cria gargalos de I/O e contenção concorrente.

### Opção 4: Snowflake / ULID

- ✅ **Ordenável no tempo:** Facilita paginação e visualização cronológica natural.
- ✅ **Descentralizado:** Não necessita de coordenação frequente de faixas.
- ❌ **Extrapola o limite de 7 caracteres:** Um ID de 64 bits codificado em Base62 gera entre 10 e 11 caracteres. Reduzir a quantidade de bits encurtaria a longevidade do sistema para menos de 5 anos.

## Links

- [Especificação do Desafio](../../CHALLENGE.md)

