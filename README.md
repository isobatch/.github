# Internet Scan Orchestrator (Batch-Oriented)

Este projeto implementa uma arquitetura determinística e auditável para varredura de larga escala da Internet, baseada em **execução por batches (netblocks fechados)**, com separação explícita entre **execução**, **coordenação**, **estado** e **observação**.

Não é um pipeline de streaming contínuo.  
É um **sistema de orquestração de scans**, alinhado ao comportamento real de ferramentas como **ZMap**, **Masscan** e **ZGrab2**.

---

## Objetivo

Executar varreduras de Internet de forma:

- controlada
- reproduzível
- semanticamente consistente
- resistente a loops implícitos e estados intermediários

Cada execução é tratada como uma **unidade discreta de trabalho (batch)**, tipicamente um **netblock**.

---

## Princípios de Design

- **Batch-first**: nada flui parcialmente.
- **Kafka como barreira**, não como cérebro.
- **Estado externo mínimo**, volátil e explícito.
- **Decisão separada de execução**.
- **Observabilidade sem acoplamento**.

---

## Conceito Central: Batch

Um **batch** representa um netblock completamente processado.

Todo evento no sistema carrega obrigatoriamente:
- `batch_id`
- `netblock`
- `asn`

Nenhum scanner publica dados intermediários.  
Um batch só existe quando **termina**.

---

## Componentes

### Scan Layer

Ferramentas reais de scanning, encapsuladas por wrappers em Rust.

#### ZMap Wrapper
- Recebe um netblock.
- Executa o scan completo.
- Define `batch_id`.
- Aplica:
  - kill-switch
  - lock por ASN/netblock
  - cooldown
- Publica **somente ao final** em `hosts.discovered`.

#### Masscan Wrapper
- Consome batches completos de hosts.
- Enumera portas.
- Respeita locks e estado.
- Publica **somente ao final** em `endpoints.discovered`.

#### ZGrab2 Wrapper
- Consome batches completos de endpoints.
- Executa fingerprint semântico.
- Publica **somente ao final** em `services.observed`.

Wrappers **não decidem prioridade**, apenas obedecem.

---

### Kafka Layer

Kafka é usado como **mecanismo de desacoplamento e sincronização**, não como sistema de decisão.

#### Tópicos principais
- `hosts.discovered`
- `endpoints.discovered`
- `services.observed`

Todos:
- chaveados por `batch_id`
- retenção curta
- sem compactação

#### Tópico de controle
- `scan.batches`
  - `batch_started`
  - `batch_completed`

Kafka garante:
- isolamento entre batches
- replay controlado
- ordem por entidade

---

### Orchestration Layer

#### n8n (Decision Point)

n8n atua exclusivamente como **orquestrador de batches**.

Funções:
- observar eventos de batch
- decidir próximos netblocks
- priorizar ou bloquear execuções
- solicitar rescan seletivo

n8n **não executa scans** e **não participa de loops de alta frequência**.

---

### State Layer

#### Redis / KV

Estado operacional volátil.

Usado para:
- lock por batch
- lock por ASN
- cooldown por netblock
- kill-switch global
- flags temporárias

Redis **não é source of truth**.

---

### Elastic Layer

Elastic é o **repositório observacional**.

- Indexação assíncrona via consumidores Kafka.
- Índices separados:
  - hosts
  - endpoints
  - services
- Todos os documentos indexados com `batch_id`.

Não existem estados intermediários no índice.

---

## Fluxo de Execução

1. n8n seleciona um netblock.
2. ZMap Wrapper executa o scan completo.
3. Ao finalizar, publica o batch em `hosts.discovered`.
4. Masscan Wrapper consome o batch inteiro.
5. Enumera portas e publica em `endpoints.discovered`.
6. ZGrab2 Wrapper consome o batch inteiro.
7. Executa fingerprint e publica em `services.observed`.
8. Elastic indexa os dados.
9. n8n observa o batch completo e decide os próximos passos.

Nenhum estágio inicia sem o batch anterior estar fechado.

---

## Propriedades do Sistema

- Determinismo forte
- Backpressure natural
- Auditabilidade por batch
- Reprocessamento simples
- Falhas previsíveis
- Sem loops implícitos

---

## Trade-offs Assumidos

- Maior latência por design
- Menos paralelismo fino
- Execução deliberadamente controlada

Esses trade-offs são intencionais.

---

## O que o sistema não é

- Não é streaming reativo
- Não é event-driven fino
- Não é auto-adaptativo
- Não é real-time

---

## Quando usar

- Scanning de larga escala
- Pesquisa de superfície de ataque
- Observação contínua da Internet
- Ambientes onde controle > velocidade

---

## Resumo

Este projeto prioriza **clareza operacional** sobre reatividade.  
Cada scan é uma decisão explícita, cada resultado é contextualizado, e cada efeito colateral é controlado.

Não escala pela abstração.  
Escala pela simplicidade correta.
