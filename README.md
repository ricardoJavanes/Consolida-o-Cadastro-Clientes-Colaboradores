# ADR 001: Consolidação do Cadastro de Clientes/Colaboradores (Ponto Único de Consulta)

## Status
Proposto

## Contexto
Atualmente, as informações cadastrais de clientes e colaboradores da Vivo coexistem e são alteradas em múltiplos sistemas, incluindo o CRM de mercado (Salesforce) e diversos sistemas legados. A área de Atendimento (CX) demanda uma visão unificada ("melhor dado") que possa ser consultada de forma centralizada. 

Os principais desafios técnicos e arquiteturais são:
1. **Baixíssima Latência:** O tempo de resposta da API de consulta precisa ser inferior a milissegundos (< 1ms para cache ou poucos milissegundos em regime de leitura).
2. **Desacoplamento Total:** O sistema de consulta não deve possuir acoplamento com as origens (Salesforce e legados).
3. **Consistência e Sincronismo:** Alterações em qualquer ponta devem refletir de forma automatizada no ponto central.
4. **Resiliência:** Falhas e indisponibilidades nos sistemas legados não podem afetar a disponibilidade da API de consulta de CX.
5. **Padronização:** A solução deve seguir o modelo "apificado" orientado aos domínios de telecomunicações.

## Decisão
Decidimos adotar uma **Arquitetura Orientada a Eventos (EDA)** baseada no padrão **CQRS (Command Query Responsibility Segregation)**, utilizando uma camada de **Change Data Capture (CDC)** para ingestão e um banco de dados NoSQL de alta performance em memória para servir como o *Golden Record Data Store*.

### Componentes Tecnológicos Escolhidos:
* **Ingestão via CDC:** Utilização do **Debezium** acoplado aos bancos de dados dos sistemas legados para capturar eventos de inserção e atualização em tempo real, sem onerar as aplicações legadas.
* **Mensageria e Streaming:** **Apache Kafka (Confluent)** como o broker central de eventos para garantir o transporte assíncrono, distribuído e persistente das alterações de dados.
* **Camada de Processamento (Workers):** Microsserviços desenvolvidos em **Spring Boot (Java)** ou **NestJS** responsáveis por consumir os tópicos do Kafka, tratar concorrências, enriquecer o dado e aplicar as regras de negócio para salvar o "melhor dado".
* **Armazenamento de Consulta (Ponto Único):** Banco de dados **Redis Enterprise** (ou **Amazon DynamoDB** com DAX) atuando como base NoSQL de chave-valor em memória.
* **Resiliência de Código:** Framework **Resilience4j** implementando padrões de *Circuit Breaker* e *Retry* nos microsserviços, associado ao uso de **Dead Letter Queues (DLQ)** no Kafka para isolamento de mensagens corrompidas.
* **Padronização de API:** Implementação baseada na especificação **TMF629 (Customer Management API)** do **TM Forum** no domínio de *Party*.

## Consequências

### Positivas (+):
* **Desempenho Extremo:** A leitura direta no Redis por chave (ID/CPF) garante tempos de resposta na casa dos microsegundos, atendendo com folga o requisito de negócio.
* **Acoplamento Zero:** Os sistemas legados e o Salesforce publicam seus dados e atualizações de forma assíncrona; eles não conhecem a existência da API de consulta e vice-versa.
* **Alta Disponibilidade e Resiliência:** Se um sistema legado ficar offline, o barramento do Kafka retém os eventos. A API de consulta de CX continua funcionando normalmente e servindo os dados já consolidados.
* **Alinhamento com Padrões Globais:** O uso das OpenAPIs do TM Forum garante conformidade com as melhores práticas mundiais de arquitetura para Telecomunicações (Telco).

### Negativas (-):
* **Consistência Eventual:** Como o fluxo de consolidação é assíncrono, há um pequeno *delay* (geralmente de milissegundos) entre a alteração no sistema de origem e o reflexo no ponto único de consulta.
* **Complexidade Operacional:** A introdução de componentes como Kafka, clusters Redis e pipelines de CDC eleva a complexidade de monitoramento (observabilidade), exigindo ferramentas robustas como Prometheus, Grafana e OpenTelemetry.

## Notas de Implementação
* O desenho detalhado dos contêineres e componentes seguirá estritamente a metodologia **C4 Model** (Níveis 1 e 2) na documentação complementar.


## Diagrama de Arquitetura

O diagrama abaixo ilustra as fronteiras da solução proposta, o desacoplamento absoluto dos sistemas legados/cloud através do barramento de eventos, e a API de consulta consumindo o repositório consolidado em memória.

![Diagrama C4 Model Agnóstico](C4-sqrc.png)


### Detalhamento dos Componentes do Desenho

# Documentação Técnica: Arquitetura de Solução Híbrida NoSQL (CQRS + EDA)

Este documento detalha a especificação técnica da **Arquitetura da Solução Target**, projetada para unificar barramentos de autenticação, ingestão assíncrona reativa de dados de sistemas legados e mecanismos de leitura em altíssima performance utilizando os padrões de **CQRS** (*Command Query Responsibility Segregation*) e **EDA** (*Event-Driven Architecture*).

---

## 🏗️ 1. Visão Geral da Arquitetura
A solução separa estritamente o fluxo transacional de escrita (sistemas de origem) do ecossistema de leitura e consulta rápida. As mutações nos sistemas legados são capturadas de forma não intrusiva via **Change Data Capture (CDC)**, transmitidas por um barramento distribuído de eventos e consolidadas de maneira assíncrona em uma base de leitura e em uma camada de cache aceleradora.

---

## 🧩 2. Detalhamento dos Componentes

### 🔐 Camada de Autenticação e Identidade
* **Keycloak (Identity Provider):** 
  Responsável pela centralização do ciclo de autenticação (login), emissão de tokens seguros padrão **JWT (JSON Web Token)** via fluxos corporativos **OAuth2 / OIDC**, gerenciamento completo de sessões de usuários e controle de consentimento. Suporta mecanismos de *Single Sign-On* (SSO).

### 🛡️ Camada de Fronteira e Governança
* **API Gateway:**
  Ponto de entrada único e síncrito para todas as requisições oriundas de aplicações externas (Web, Mobile ou Clientes Backend).
* **Authorizer (JWT):**
  Filtro acoplado de forma nativa ao gateway que realiza a inspeção e validação criptográfica em tempo de execução de assinaturas JWT, verificação de expiração do token, validação de emissores (*issuer/audience*) e extração de escopos ou restrições (*claims*).
* **Outras Proteções:**
  O gateway encapsula módulos de **WAF** (Web Application Firewall) contra vulnerabilidades estruturais, políticas de limitação de taxa (**Rate Limiting**) para mitigar abusos de infraestrutura, além de centralizar a coleta de logs e métricas operacionais.

### ⚙️ Camada de Aplicação e Processamento (Read Context)
* **BFF (Backend For Frontend):**
  Microsserviço de borda responsável pela orquestração de chamadas internas, formatação de payloads otimizados para telas específicas, tratamento de contextos e aplicação de regras de usabilidade. A comunicação ocorre de forma estrita via segurança de canal (HTTPS/TLS e mTLS).
* **API de Negócio:**
  Microsserviço agnóstico de canais encarregado de aplicar regras de autorização de negócio detalhadas, validação de permissões finas por recurso ou dado solicitado, controle estrito de acesso granular às coleções e geração de registros de auditoria em conformidade com as políticas corporativas.

### 📥 Camada de Ingestão de Dados (Write Model Context)
* **Sistemas Legados (Sistemas A, B e C):**
  Repositórios transacionais relacionais que mantêm seus bancos de dados próprios isolados em silos produtivos.
* **Debezium (Connector CDC):**
  Mecanismo de captura baseado em logs de transação ativas (WAL/Redo). Ele escuta, traduz e despacha todas as operações de mutação (`INSERT`, `UPDATE`, `DELETE`) ocorridas nas tabelas mapeadas sem executar queries síncronas que onerem as instâncias de CPU dos legados.

### 📭 Camada de Streaming e Mensageria (EDA Core)
* **Apache Kafka (Cluster):**
  Espinha dorsal distribuída e de alta disponibilidade responsável pela ingestão contínua e imutável dos fluxos de dados reorganizados em tópicos segmentados por entidades de negócio (`clientes.changed`, `pedidos.changed`, `produtos.changed`).

### 🔄 Camada de Sincronização e Destino (Kafka Connect Cluster)
Módulo distribuído encarregado de processar concorrentemente os tópicos de mensagens e executar a automação de escrita reativa nas bases NoSQL:
* **Redis Sink Connector:**
  Consome o Kafka em tempo real, transforma os esquemas de dados em estruturas chave-valor rápidas e atualiza/invalida o cache dinamicamente em lote (`SET / HSET / DEL`).
* **MongoDB Sink Connector:**
  Consome concorrentemente os mesmos fluxos de mensagens, consolida visões aninhadas e executa operações de escrita documental estruturada via comandos nativos de *upsert* ou eliminação de chaves.

### 💾 Camada de Armazenamento e Leitura (Read Model / Cache)
* **Redis (Cache):**
  Armazenamento NoSQL em memória de altíssima performance. Mantém dados temporários indexados com ciclo de vida atrelado a políticas rigorosas de expiração (TTL), garantindo respostas síncronas em escala microscópica (< 1ms).
* **MongoDB (Read Model):**
  Banco de dados NoSQL baseado em documentos orientados a JSON. Atua como o repositório durável consolidado (*Golden Record*), guardando o histórico unificado completo do cliente para viabilizar consultas complexas ou agregadas que o cache chave-valor não suporta nativamente.

---

## 🔄 3. Detalhamento dos Fluxos Operacionais

### 🧵 Fluxo Principal (Consulta da API)
1. O usuário final ou aplicação cliente realiza a autenticação segura no **Keycloak**.
2. O **Keycloak** valida as credenciais e emite um token assinado **JWT (Access Token)**.
3. O cliente submete a requisição HTTPS (TLS) chamando o endpoint exposto no **API Gateway**.
4. O **API Gateway** aciona o componente **Authorizer**, que inspeciona o token de borda.
5. Sendo validado e autorizado, o tráfego é roteado internamente via segurança mTLS para o **BFF**.
6. O **BFF** formata a lógica e invoca a **API de Negócio**.
7. A **API** valida as permissões de acesso ao recurso, executa a auditoria e direciona a leitura para o **Redis (Cache)**:
   * **Cenário de Cache Hit (Dados Encontrados):** O Redis retorna instantaneamente os dados cadastrais rápidos para a API, que os encaminha de volta para o cliente final, finalizando o fluxo.
   * **Cenário de Cache Miss (Dados Não Encontrados):** Caso a informação não esteja no cache local, a API dispara uma busca imediata contra o **MongoDB (Read Model)**. O MongoDB retorna a estrutura de documentos JSON completa para a API responder ao usuário. Opcionalmente, um gatilho assíncrono sincroniza e popula o Redis para otimizar os acessos sequenciais do mesmo ID.

### ⚙️ Fluxo de Dados de Sincronização (CDC + Kafka + Connectors)
1. Mudanças de estado de dados ocorrem isoladamente no banco de um **Sistema Legado**.
2. O **Debezium** detecta a alteração de log e publica de forma assíncrona uma mensagem no **Apache Kafka**.
3. O cluster do **Kafka** disponibiliza os pacotes estruturados no tópico por domínio específico.
4. Os conectores especializados do **Kafka Connect** leem de forma agnóstica e paralela a fila:
   * O **Redis Sink Connector** atua em memória atualizando as chaves ativas do cache.
   * O **MongoDB Sink Connector** executa rotinas de *upsert* no banco de leitura completo.

---

## 📈 4. Principais Benefícios da Solução
* **Dados Sempre Atualizados:** O pipeline reativo assíncrono (CDC + Kafka) garante replicação e convergência em tempo quase real.
* **Leitura Ultraveloz:** Divisão explícita de performance utilizando Redis para baixa latência de chave única e MongoDB para robustez documental estruturada.
* **Segurança Profunda:** Modelo de blindagem em camadas integrado do Keycloak até a padronização de mTLS entre microsserviços.
* **Resiliência Arquitetural:** Em caso de falhas na camada transacional das fontes originais, a camada de atendimento continua ativa consumindo as bases lógicas de leitura desacopladas sem causar impactos nas operações.

---
*Uso Interno e Confidencial - Arquitetura de Solução Integrada*


