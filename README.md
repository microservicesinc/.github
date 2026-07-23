# Microservices Architecture Lab - Showcase 🚀

Este espacio ha sido diseñado para demostrar la implementación de una arquitectura híbrida de microservicios, combinando cómputo tradicional en contenedores con un diseño moderno **Serverless orientado a eventos**. 

El ecosistema simula un caso de uso común de negocio: **Gestión de Órdenes de Compra e Inventario**, priorizando el desacoplamiento, la resiliencia y la escalabilidad automática de componentes específicos.

---

## 🎯 Intención del Proyecto

El objetivo principal es resolver el problema del acoplamiento temporal entre servicios críticos mediante mensajería asíncrona. 

1. **Cómputo Core Estable:** Las APIs principales de negocio corren sobre contenedores tradicionales dentro de una infraestructura dedicada (**AWS EC2**), asegurando latencias predecibles.
2. **Operaciones Atómicas Scalables:** Las tareas secundarias desencadenadas por el negocio (como el procesamiento de stock posterior a una venta) se delegan a funciones **AWS Lambda**, eliminando carga computacional de los servidores principales y optimizando costos.
3. **Punto de Entrada Unificado:** Toda la comunicación externa con los clientes se gestiona a través de una capa de API Gateway, abstrayendo la topología interna de la red.

---

## 🗺️ Roadmap de Implementación

El proyecto se ejecuta de forma incremental a través de las siguientes fases estratégicas para asegurar la calidad del código y la automatización:

### ⚙️ Fase 1: Simulación y Desarrollo Local
* **Objetivo:** Construir la lógica de negocio a costo cero y validar el comportamiento reactivo antes de ir a producción.
* **Tecnologías:** Docker, Docker Compose, LocalStack (emulación local de DynamoDB, SNS y SQS) y Nginx (como API Gateway local).
* **Entregable:** Un flujo local verificado vía Postman donde el servicio de Órdenes (.NET Core 10) publica eventos que impactan al servicio de Inventario (Python/Flask) sin conexión HTTP directa.

### 📜 Fase 2: Infraestructura como Código (IaC)
* **Objetivo:** Automatizar la creación de todo el entorno cloud real para evitar configuraciones manuales propensas a errores.
* **Tecnologías:** AWS CDK (Cloud Development Kit).
* **Entregable:** Scripts declarativos en el repositorio de infraestructura que aprovisionan bases de datos NoSQL (DynamoDB), la VPC de red, servidores EC2, tópicos/colas (SNS/SQS) y el mapeo de eventos hacia la AWS Lambda.

### ☁️ Fase 3: Despliegue Cloud y Showcase Técnico
* **Objetivo:** Migrar la arquitectura verificada a un entorno real productivo utilizando créditos de capa gratuita de AWS.
* **Tecnologías:** Amazon API Gateway, AWS EC2, Amazon DynamoDB, AWS Lambda.
* **Entregable:** Demostración de punta a punta con URLs públicas de AWS Gateway, monitoreo de colas y persistencia NoSQL de baja latencia en vivo.

---

## 📂 Estructura de Repositorios

* **`orders-service`**: Microservicio encargado del ciclo de vida de las compras (.NET Core 10).
* **`inventory-service`**: Microservicio que gestiona el stock de productos (Python/Flask) y aloja el código de la Lambda atómica.
* **`infrastructure`**: Código fuente de AWS CDK para el despliegue automatizado de la arquitectura en la nube.

## Diseño 

```mermaid
graph TD
    %% Repositorios de Código
    subgraph Repositorios_GitHub [Organización GitHub: microservicesinc]
        R1["<b>Repo 1: Orders Service</b><br/>• Web API (.NET Core 10)<br/>• Código Lambda (Notificaciones)"] 
        R2["<b>Repo 2: Inventory Service</b><br/>• Flask API (Python)<br/>• Código Lambda (Stock Worker)"]
        R3["<b>Repo 3: Infrastructure</b><br/>• AWS CDK (Crea EC2, SNS, SQS, Lambdas)"]
    end

    %% Entrada y Cómputo Fijo (EC2)
    Cliente((Cliente / Postman)) -->|HTTP Requests| APIGW[<b>Amazon API Gateway</b><br/>Proxy / Reverso]
    
    subgraph VPS_EC2 [Instancia AWS EC2 - Cómputo Tradicional]
        DockerCompose["<b>Docker Compose</b>"]
        ContainerOrders["<b>Contenedor: Orders API</b><br/>(.NET Core 10)"]
        ContainerInventory["<b>Contenedor: Inventory API</b><br/>(Flask)"]
        
        DockerCompose --> ContainerOrders
        DockerCompose --> ContainerInventory
    end

    APIGW -->|/orders| ContainerOrders
    APIGW -->|/inventory| ContainerInventory

    %% Persistencia
    ContainerOrders -->|Escritura Sincrónica| DynamoOrders[("<b>DynamoDB</b><br/>Tabla Órdenes")]

    %% Eventos Asíncronos
    ContainerOrders -->|Publica evento 'OrderPlaced'| SNSTopic[["<b>Amazon SNS</b><br/>Tópico de Eventos"]]
    SNSTopic -->|Fan-out / Suscripción| SQSQueue[["<b>Amazon SQS</b><br/>Cola de Mensajes"]]

    %% Lambdas Atómicas (Serverless)
    subgraph AWS_Lambda_Serverless [Tareas Atómicas e Independientes]
        LambdaStock["<b>AWS Lambda: Stock Worker</b><br/>(Escucha la cola SQS)"]
        LambdaNotify["<b>AWS Lambda: Notification</b><br/>(Ej: Envía Email de Confirmación)"]
    end

    SQSQueue -->|Polling Automático| LambdaStock
    LambdaStock -->|Actualiza Stock de Forma Segura| DynamoInventory[("<b>DynamoDB</b><br/>Tabla Inventario")]
    
    SNSTopic -->|Suscripción Directa| LambdaNotify

    %% Mapeo de Código e IaC
    R1 -.->|Código API| ContainerOrders
    R1 -.->|Código Handler| LambdaNotify
    R2 -.->|Código API| ContainerInventory
    R2 -.->|Código Handler| LambdaStock
    R3 -.->|Despliega Infraestructura| VPS_EC2
    R3 -.->|Crea Recursos Serverless| AWS_Lambda_Serverless

    %% Estilos Visuales
    style Repositorios_GitHub fill:#f9f9f9,stroke:#333,stroke-width:2px
    style R1 fill:#fff7e6,stroke:#ff9900,stroke-width:2px
    style R2 fill:#fff7e6,stroke:#ff9900,stroke-width:2px
    style R3 fill:#e6f7ff,stroke:#1890ff,stroke-width:2px
    style VPS_EC2 fill:#f5f5f5,stroke:#7f8c8d,stroke-width:2px,stroke-dasharray: 5 5
    style ContainerOrders fill:#e8f4f8,stroke:#2980b9,stroke-width:2px
    style ContainerInventory fill:#e8f4f8,stroke:#2980b9,stroke-width:2px
    style AWS_Lambda_Serverless fill:#ebf5fb,stroke:#27ae60,stroke-width:2px
```
