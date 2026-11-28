# 2. Objetivos

## 2.1 Objetivo geral

Construir um sistema de monitoramento de risco de transações de pagamento que avalie cada transação e decida **APROVAR**, **REVISAR** ou **BLOQUEAR** dentro de um tempo limite pré-definido, e que ofereça a analistas de fraude ferramentas de investigação baseadas em grafos, mantendo os dados relacionais (PostgreSQL) e o grafo (Neo4j) sincronizados de forma automática e confiável.

## 2.2 Objetivos específicos

### Avaliação de risco

- **OE1.** Implementar um motor de regras que calcule um *score* de risco por transação e traduza esse score em uma decisão (APROVAR, REVISAR ou BLOQUEAR).
- **OE2.** Garantir que a avaliação respeite um tempo limite (*timeout*); se o tempo se esgotar, o motor deve devolver uma decisão padrão definida no catálogo de regras, e nunca ficar sem resposta.
- **OE3.** Expor a avaliação por meio de uma API REST.

### Investigação de fraudes

- **OE4.** Modelar as relações entre clientes, contas, dispositivos e transações em um grafo no Neo4j.
- **OE5.** Disponibilizar consultas de investigação para três padrões de fraude: contas que compartilham dispositivos, ciclos de transferência e redes de contas-laranja.
- **OE6.** Oferecer ao analista uma fila de revisão e uma tela de investigação da rede no aplicativo desktop.

### Integração e consistência dos dados

- **OE7.** Manter PostgreSQL e Neo4j sincronizados por meio de eventos publicados em Redis Pub/Sub e processados por rotas Apache Camel, usando um modelo canônico e um mapeamento de identificadores entre os dois bancos.
- **OE8.** Garantir que o integrador seja **idempotente** (o mesmo evento processado mais de uma vez não altera o resultado) e que trate falhas sem perder eventos.
- **OE9.** Sincronizar de volta para o PostgreSQL as decisões de risco calculadas no Neo4j/API.

### Qualidade e processo

- **OE10.** Medir a latência da avaliação com um simulador de transações e validar o sistema com testes unitários, de integração e ponta a ponta.
- **OE11.** Documentar o projeto (README, JavaDoc, Swagger, diagramas) e seguir um fluxo de trabalho rastreável: issues, branches, pull requests revisados e commits pequenos.

## 2.3 Metas mensuráveis

| Meta | Critério de aceite |
|---|---|
| Tempo de avaliação | Avaliação concluída dentro do tempo limite definido no `risk-engine`, medido pelo simulador |
| Decisão sempre presente | 100% das requisições recebem APROVAR, REVISAR ou BLOQUEAR, inclusive em caso de timeout |
| Consistência entre bancos | Após o processamento dos eventos, cada registro do PostgreSQL tem seu correspondente no Neo4j (mesmo ID canônico) |
| Idempotência | Reenviar um evento já processado não cria duplicatas nem altera o estado |
| Detecção de fraude | Os anéis de fraude do gerador de massa de dados são encontrados pelas consultas de dispositivos compartilhados, ciclos e contas-laranja |

## 2.4 Escopo

**Dentro do escopo:** cadastro e back-office (desktop), avaliação de risco (API + motor de regras), consultas de investigação no grafo, integração PostgreSQL ↔ Neo4j.

**Fora do escopo:** integração com meios de pagamento reais, autenticação e autorização em produção, modelos de aprendizado de máquina para fraude. A análise com Neo4j GDS (comunidades e PageRank) é opcional.
