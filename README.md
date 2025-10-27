#  Projeto de Microsserviços — Warehouse & Storefront

##  Descrição

Este projeto demonstra uma **arquitetura de microsserviços em Java (Spring Boot)** composta por **dois serviços independentes** que se comunicam de forma:

- **Síncrona** → via HTTP (REST)
- **Assíncrona** → via RabbitMQ (mensageria)

### Serviços

| Serviço | Porta | Descrição |
|----------|--------|-----------|
|  **Warehouse** | `8081` | Serviço de **armazém**. Gerencia o estoque de produtos e publica eventos de atualização. |
|  **Storefront** | `8080` | Serviço de **vitrine**. Consulta o estoque via HTTP e consome eventos RabbitMQ para manter o cache atualizado. |

---

##  Tecnologias utilizadas

- **Java 17**
- **Spring Boot 3.x**
- **Spring Web / WebFlux**
- **Spring Data JPA / H2**
- **Spring AMQP (RabbitMQ)**
- **Docker / Docker Compose**
- **Maven**

---

##  Arquitetura Geral

    +---------------------------+
    |       Warehouse (8081)    |
    |---------------------------|
    | - Armazena estoque        |
    | - API REST /warehouse/... |
    | - Publica eventos Rabbit  |
    +-------------+-------------+
                  |
     HTTP (GET)   |    Mensagem (RabbitMQ)
                  |
    +-------------v-------------+
    |       Storefront (8080)   |
    |---------------------------|
    | - Consulta estoque via HTTP |
    | - Mantém cache via eventos |
    | - Exibe dados da vitrine   |
    +---------------------------+

---

##  Executando o projeto

###  Pré-requisitos

- **Java 17+**
- **Maven 3.8+**
- **Docker** (para RabbitMQ)

---

 Passo 1 — Subir o RabbitMQ

Inicie o broker RabbitMQ via Docker:

docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management

 Passo 2 — Rodar os microsserviços

Abra dois terminais diferentes:

- Warehouse (porta 8081)

cd warehouse
mvn clean spring-boot:run

- Storefront (porta 8080)

cd storefront
mvn clean spring-boot:run

 Passo 3 — Testar endpoints

Ajustar o estoque (gera evento RabbitMQ)
curl -X POST "http://localhost:8081/warehouse/products/1/adjust?delta=10"

Ver disponibilidade no Warehouse (síncrono)
curl http://localhost:8081/warehouse/products/1/availability

Ver disponibilidade no Storefront (HTTP + cache assíncrono)
curl http://localhost:8080/storefront/products/1/availability

Ver cache local da vitrine (mantido via RabbitMQ)
curl http://localhost:8080/storefront/cache

 Estrutura do projeto
 /warehouse
 ├─ pom.xml
 ├─ Dockerfile
 └─ src/main/java/com/example/warehouse/
     ├─ WarehouseApplication.java
     ├─ config/RabbitConfig.java
     ├─ dto/
     ├─ domain/
     ├─ repository/
     ├─ service/
     └─ web/
 /storefront
 ├─ pom.xml
 ├─ Dockerfile
 └─ src/main/java/com/example/storefront/
     ├─ StorefrontApplication.java
     ├─ config/
     ├─ dto/
     ├─ listener/
     ├─ service/
     └─ web/
 docker-compose.yml
 README.md

---

##  Comunicação via RabbitMQ

### Exchange / Queue configurados

| Tipo         | Nome                 | Descrição                                      |
|---------------|----------------------|------------------------------------------------|
| **Exchange**  | `stock.exchange`     | Tópico principal de mensagens de estoque       |
| **Queue**     | `stock.updated.queue`| Fila onde o `storefront` ouve as atualizações  |
| **Routing key** | `stock.updated`    | Rota usada para enviar eventos de estoque      |

---

 Fluxo de funcionamento

- O warehouse recebe uma requisição POST /warehouse/products/{id}/adjust.

- Atualiza o estoque no banco (ProductStock).

- Publica um evento StockUpdateEvent via RabbitMQ.

- O storefront escuta a fila stock.updated.queue e atualiza seu cache local.

- Quando o storefront faz uma consulta HTTP ao warehouse, ele pode comparar o dado do cache com o retorno real.
