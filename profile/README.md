Pipeline distribuído para descoberta de hosts, identificação de serviços e observação estruturada, orientado a eventos e baseado em Kafka como log central imutável.

O sistema é composto por **aplicações Rust independentes**, desacopladas por tópicos Kafka, permitindo escala seletiva, replay completo e substituição isolada de estágios.

---

## Visão Geral da Arquitetura

O pipeline é **estritamente unidirecional**. Cada aplicação executa uma única responsabilidade, publica eventos e não mantém estado interno persistente.

Fluxo lógico: hosts → endpoints → fingerprint → services → elastic


Kafka atua como **barreira arquitetural**, não como fila.

---

## Componentes

### Discovery Applications

#### `hosts-discovery`
- Responsável por descoberta de hosts (ex: masscan).
- Publica eventos em `hosts.discovered`.

#### `ports-discovery`
- Consome `hosts.discovered`.
- Descobre portas e protocolos expostos.
- Publica eventos em `endpoints.discovered`.

---

### Service Identification Applications

#### `fingerprint`
- Consome `endpoints.discovered`.
- Executa probes e fingerprinting de protocolos.
- Publica eventos em `fingerprint.detected`.

#### `negotiate`
- Consome `fingerprint.detected`.
- Realiza negociação ativa (handshakes, banners, variações de protocolo).
- Publica eventos finais em `services.observed`.

---

### Observer Application

#### `service-observer`
- Consome `services.observed`.
- Responsável por:
  - normalização
  - enrichment
  - deduplicação
  - transformação em documento
- Indexa dados no Elastic/OpenSearch.
- Pode ser desligado, reprocessado ou substituído sem impacto no scan.

---

## Kafka Topics

| Topic                  | Descrição                                | Key recomendada                |
|------------------------|------------------------------------------|--------------------------------|
| `hosts.discovered`     | Hosts identificados                      | `ip`                           |
| `endpoints.discovered` | Endpoints (ip/porta/protocolo)           | `ip:port:proto`                |
| `fingerprint.detected` | Serviço identificado                     | `ip:port:service`              |
| `services.observed`    | Serviço observado e negociado            | `service_fingerprint_hash`     |

Todos os eventos devem ser:
- imutáveis
- versionados por schema
- idempotentes por key

---

## Execução

Cada aplicação é executada de forma independente.

Exemplos:

```bash
KAFKA_BROKERS=190.102.43.107:9092 target/release/hosts-discovery 200.150.192.0/20
KAFKA_BROKERS=190.102.43.107:9092 target/release/ports-discovery
KAFKA_BROKERS=190.102.43.107:9092 target/release/fingerprint
KAFKA_BROKERS=190.102.43.107:9092 target/release/negotiate
KAFKA_BROKERS=190.102.43.107:9092 target/release/service-observer
```

## Princípios Arquiteturais

- Nenhuma comunicação direta entre aplicações  
- Nenhum estado compartilhado em memória  
- Kafka é a única fonte de verdade  
- Elastic não participa do pipeline de scan  
- Todo estágio é escalável de forma independente  
- Falha parcial não invalida o pipeline  
- Replay completo é sempre possível  

---

## O que este projeto **não é**

- Não é uma ferramenta monolítica de scan  
- Não é um conjunto de microserviços HTTP  
- Não depende de RPC, REST ou gRPC  
- Não acopla lógica de negócio a storage  

---

## Objetivo

Construir um pipeline de observação distribuída, determinístico e reprocessável, capaz de operar em larga escala com isolamento funcional real.

Este projeto prioriza:

- Clareza arquitetural  
- Previsibilidade operacional  
- Controle total sobre fluxo, estado e evolução  
