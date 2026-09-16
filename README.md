# ADR 001: Consolidação do Cadastro de Clientes/Colaboradores (Ponto Único de Consulta)

## 1. Status
Proposto

## 2. Contexto
Atualmente, as informações cadastrais de clientes e colaboradores da Vivo coexistem e são alteradas em múltiplos sistemas, incluindo o CRM de mercado (Salesforce), sistemas em nuvem e diversos sistemas legados on-premises. A área de Atendimento (CX) demanda uma visão unificada ("melhor dado" / Visão 360°) que possa ser consultada de forma centralizada a partir de canais como o App Meu Vivo e o CRM do Agente. 

Os principais desafios técnicos e arquiteturais são:
1. **Baixíssima Latência:** O tempo de resposta da API de consulta precisa ser inferior a milissegundos (< 5ms no Cache Hit e < 15ms no fallback).
2. **Desacoplamento Total:** O sistema de consulta principal não deve possuir acoplamento com as origens (sistemas legados e cloud não devem "se conhecer").
3. **Consistência e Sincronismo:** Alterações em qualquer ponta devem refletir de forma automatizada e em tempo quase real no ponto único de consulta.
4. **Resiliência:** Falhas, instabilidades e indisponibilidades nos sistemas legados não podem indisponibilizar ou afetar a performance da API de consulta de CX.
5. **Padronização:** A solução deve ser totalmente "APIFICADA", seguindo o modelo orientado aos domínios e contratos globais de telecomunicações.

## 3. Decisão
Decidimos adotar uma **Arquitetura Orientada a Eventos (EDA)** baseada no padrão **CQRS (Command Query Responsibility Segregation)**. Toda a ingestão será realizada via **Change Data Capture (CDC)** de forma não intrusiva, alimentando um modelo híbrido de leitura composto por um repositório NoSQL e um Cache Distribuído em memória (Single Source of Truth).

### Componentes Tecnológicos Escolhidos:
* **Camada de Entrada e Governança:** Utilização de um **API Gateway (WSO2 API Manager / Kong)** atuando como ponto único de entrada desacoplado, tratando autenticação (OAuth2/OIDC), rate limiting e governança.
* **Ingestão via CDC:** Conectores **Debezium (Kafka Connect)** lendo diretamente os logs de transação (Redo/WAL) dos bancos legados (Oracle/Mainframe) e Cloud (PostgreSQL), sem realizar queries nas tabelas de produção.
* **Mensageria e Streaming:** Cluster **Apache Kafka** como barramento de eventos assíncrono para garantir o transporte distribuído, resiliente e persistente das mensagens.
* **Processamento e Sincronização:** O microsserviço **Customer-Sync-Worker** consumirá os tópicos do Kafka, aplicando regras de negócio, enriquecimento, merge e deduplicação cadastral por CPF/CNPJ.
* **Repositório Unificado (Read Model):** Combinação híbrida entre **MongoDB / Amazon DocumentDB** (para armazenar a visão unificada estruturada do cliente em JSON / 360°) e **Redis Cluster** (como cache em memória operando em padrão *Write-Through / Cache-Aside* para respostas abaixo de 5ms).
* **Resiliência:** Implementação do framework **Resilience4j** (*Circuit Breaker* e *Retry*) nas aplicações, isolamento de mensagens inconsistentes via **Dead Letter Queues (DLQ)** no Kafka, e mecanismos de fallback automático no banco NoSQL em caso de *Cache Miss*.
* **Padronização de API:** Modelagem baseada nas especificações OpenAPIs da **TM Forum**, utilizando os contratos canônicos dos domínios necessários.

## 4. Diagrama da Arquitetura de Solução (C4 Model - Nível 2: Contêineres)

O diagrama abaixo ilustra a separação física e lógica em camadas da solução, evidenciando o isolamento completo dos legados por meio da mensageria e o fluxo síncrono de altíssima performance para os canais.

```mermaid
graph TD
    %% Estilos Globais
    classDef canais fill:#ffffff,stroke:#00a1e4,color:#000,stroke-width:2px;
    classDef exposicao fill:#ffffff,stroke:#006699,color:#000,stroke-width:2px;
    classDef consulta fill:#e8f5e9,stroke:#2e7d32,color:#000,stroke-width:2px;
    classDef ingestao fill:#fff3e0,stroke:#e65100,color:#000,stroke-width:2px;
    classDef origens fill:#f5f5f5,stroke:#37474f,color:#000,stroke-width:2px;
    classDef db fill:#ffffff,stroke:#1b5e20,color:#000,stroke-width:1px;
    classDef kafka fill:#ffffff,stroke:#bf360c,color:#000,stroke-width:1px;

    %% CAMADA DE CANAIS / CONSUMIDORES
    subgraph CamadaCanais [CAMADA DE CANAIS / CONSUMIDORES]
        MeuVivo[App Meu Vivo <br><i>(Mobile Native)</i>]:::canais
        CrmAgente[CRM do Agente / Web Portal <br><i>(Web Application)</i>]:::canais
    end

    %% CAMADA DE EXPOSIÇÃO E GOVERNANÇA
    subgraph CamadaExposicao [CAMADA DE EXPOSIÇÃO E GOVERNANÇA (API MANAGEMENT)]
        ApiGateway[WSO2 API Manager / Kong Gateway <br><b>[OAuth2 / OIDC | Rate Limiting | Open API TM Forum]</b>]:::exposicao
    end

    %% CAMADA DE CONSULTA (READ MODEL)
    subgraph CamadaConsulta [CAMADA DE CONSULTA (READ MODEL)]
        QueryApi[Customer-Query-API <br><i>(Spring Boot / Go - Microservice)</i>]:::consulta
        Redis[Redis Cluster <br><i>(In-Memory Cache < 5ms)</i>]:::db
        Mongo[MongoDB / DocumentDB <br><i>(Single View 360°)</i>]:::db
    end

    %% CAMADA DE INGESTÃO E EVENTOS (EDA)
    subgraph CamadaIngestao [CAMADA DE INGESTÃO E EVENTOS (EDA)]
        SyncWorker[Customer-Sync-Worker <br><i>(Merge & Deduplication Service)</i>]:::ingestao
        Kafka[Apache Kafka Cluster <br><i>(Topics: customer.events / DLQ)</i>]:::kafka
        CdcLegados[Debezium CDC <br><i>(Legados)</i>]:::ingestao
        CdcCloud[Debezium CDC <br><i>(Cloud Services)</i>]:::ingestao
    end

    %% SISTEMAS LEGADOS e CLOUD
    subgraph SistemasLegados [SISTEMAS LEGADOS (ON-PREMISES)]
        OracleDB[Oracle DB / Mainframe <br><i>(Reads WAL / Redo Logs)</i>]:::origens
    end

    subgraph SistemasCloud [SISTEMAS CLOUD]
        PostgresCloud[PostgreSQL / Cloud Services / Salesforce <br><i>(Reads Change Logs)</i>]:::origens
    end

    %% Relacionamentos e Fluxos
    MeuVivo -->|HTTPS / REST| ApiGateway
    CrmAgente -->|HTTPS / REST| ApiGateway
    
    ApiGateway -->|TMF629 / TMF632| QueryApi
    
    QueryApi -->|Cache Hit < 5ms| Redis
    QueryApi -->|Cache Miss / Fallback| Mongo
    
    SyncWorker -.->|Update Async| Redis
    SyncWorker -.->|Update Async| Mongo
    SyncWorker -->|Consume Topics| Kafka
    
    Kafka <--|CDC Events| CdcLegados
    Kafka <--|CDC Events| CdcCloud
    
    CdcLegados -.->|Reads Redo/WAL Logs| OracleDB
    CdcCloud -.->|Reads Change Logs| PostgresCloud
```

## 5. Consequências

### Positivas (+):
* **Desempenho Sub-Milissegundo:** Leituras diretas via chaves no Redis garantem latências extremamente baixas (< 5ms) para a camada de CX.
* **Isolamento de Carga (Offloading):** Os sistemas legados on-premises e cloud ficam protegidos contra picos de tráfego de consulta gerados pelos canais digitais.
* **Acoplamento Zero:** As origens apenas disponibilizam as alterações em logs transacionais assíncronos. Seus ciclos de vida permanecem 100% independentes do microsserviço de consulta.
* **Resiliência a Falhas:** Caso um sistema de origem falhe, a fila do Kafka retém os eventos gerados. A aplicação de consulta continua respondendo normalmente com a última versão consolidada e estável em cache.
* **Conformidade Global e LGPD:** A padronização de contratos via TM Forum unifica os dados do titular e resolve problemas legais de duplicidade de perfis.

### Negativas (-):
* **Consistência Eventual:** Devido ao fluxo de processamento assíncrono e baseado em streaming de eventos, existe um delay em milissegundos para que uma escrita em um legado reflita no modelo de leitura.
* **Complexidade de Infraestrutura:** Exige maior governança e monitoramento de novos componentes distribuídos (Kafka Connect, Brokers, Clusters NoSQL e Redis), demandando ferramentas sólidas de observabilidade (OpenTelemetry, Prometheus e Grafana).

## 6. Governança & Padronização TM Forum APIs

A adoção dos padrões abertos da **TM Forum** resolve o desafio de fragmentação de dados no ecossistema de Telecom, fornecendo contratos REST/JSON universais que desconectam o front-end dos esquemas legados proprietários.

| API TM Forum | Domínio / Responsabilidade | Recursos & Função Prática |
| :--- | :--- | :--- |
| **TMF629**<br>Customer Management | Papel Comercial do Cliente (Customer Role) | Recurso `/customer`. Gerencia status da conta (Active/Suspended), preferências de contato e vínculo com contas de faturamento. |
| **TMF632**<br>Party Management | Entidade Mestra Real (Individual / Organization) | Recursos `/individual` e `/organization`. Mantém CPF/CNPJ, nome, data de nascimento e documentos oficiais. Base MDM. |
| **TMF637**<br>Product Inventory | Produtos e Serviços Contratados | Recurso `/product`. Mapeia planos, linhas móveis, banda larga, chips e status do catálogo ativo. |
| **TMF666**<br>Account Management | Contas de Faturamento & Cobrança | Recurso `/billingAccount`. Dados de ciclo de faturamento, histórico de faturas e limite de crédito. |

## 7. Mapeamento Técnico de Atributos (Modelo Canônico TMF629)

Para materializar o conceito de "melhor dado" (*Golden Record*) no ponto único de consulta, o **Customer-Sync-Worker** realiza o mapeamento e a higienização dos payloads brutos transformando-os na estrutura canônica da **TMF629**.

### Objeto Principal: `Customer`
*   **`id`** (String): Identificador único global do cliente gerado de forma determinística por hash (ex: baseado no CPF/CNPJ) para evitar duplicidade entre origens.
*   **`href`** (String): URL de auto-referência para acesso direto ao recurso da API (ex: `https://vivo.com.br`).
