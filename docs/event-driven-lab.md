# Laboratorio: Creación de Microservicios con Spring Boot, RabbitMQ, Docker y Play With Docker

**Objetivo**: Crear dos servicios Spring Boot (Productor y Consumidor) que se comuniquen a través de RabbitMQ, todo orquestado con Docker Compose para facilitar las pruebas en Play With Docker.

Estructura del Repositorio (Ejemplo):

```
event-driven-lab/
├── producer-service/         # Servicio Productor Spring Boot
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── consumer-service/         # Servicio Consumidor Spring Boot
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── .devcontainer/            # Configuración para Codespaces
│   └── devcontainer.json
├── docker-compose.yml        # Orquestación para Play with Docker
└── README.md                 # Instrucciones
```

## Paso 1: Configurar GitHub Codespaces

1. Crea un Repositorio en GitHub: Crea un nuevo repositorio público o privado llamado ```event-driven-lab``` (o el nombre que prefieras).

2. Añade la Configuración de .devcontainer:
* Dentro de tu repositorio, crea una carpeta .devcontainer.
* Dentro de .devcontainer, crea un archivo devcontainer.json con el siguiente contenido:

```json
{
    "name": "Java Spring Event Lab",
    // Usa una imagen base con Java y Maven preinstalados.
    // Puedes ajustar la versión de Java (e.g., :11, :17, :21)
    "image": "mcr.microsoft.com/devcontainers/java:0-17",

    "features": {
        // Incluye Docker-in-Docker para poder construir imágenes dentro de Codespaces si es necesario
        // Aunque el despliegue final es en Play with Docker, puede ser útil para pruebas locales.
        "ghcr.io/devcontainers/features/docker-in-docker:2": {},
        // Añade explícitamente la feature de Maven
        "ghcr.io/devcontainers/features/java:1": {
          "version": "none",
          "installMaven": "true",
          "mavenVersion": "3.8.6",
          "installGradle": "false"
        }
    },

    // Puertos a exponer desde Codespaces (útil para pruebas locales)
    "forwardPorts": [
        8080, // Puerto del productor
        8081, // Puerto del consumidor (si tuviera API REST)
        15672, // RabbitMQ Management UI
        5672  // RabbitMQ AMQP Port
    ],

    // Extensiones de VS Code recomendadas
    "customizations": {
        "vscode": {
            "extensions": [
                "vscjava.vscode-java-pack",
                "vmware.vscode-spring-boot",
                "pivotal.vscode-spring-boot",
                "ms-azuretools.vscode-docker",
                "redhat.java",
                "pivotal.vscode-boot-dev-pack",
                "vmware.vscode-boot-dev-pack"
            ]
        }
    },

    // Comando a ejecutar después de crear el contenedor (opcional, ej: instalar algo)
    // "postCreateCommand": "mvn --version",

    // Configura el usuario remoto
    "remoteUser": "vscode"
}
```

3. **Inicia Codespaces**: Ve a tu repositorio en GitHub, haz clic en el botón <> Code, selecciona la pestaña "Codespaces" y haz clic en "Create codespace on main". GitHub preparará tu entorno de desarrollo basado en devcontainer.json.

---

## Paso 2: Crear el Servicio Productor (Spring Boot)

1. Navega a la Carpeta Raíz en Codespaces: Asegúrate de estar en /workspaces/event-driven-lab en la terminal de Codespaces.

2. Crea el Proyecto Productor: Puedes usar Spring Initializr (start.spring.io) o el asistente de VS Code.
* Dependencias:
    * Spring Web (para crear un endpoint REST para enviar mensajes)
    * Spring for RabbitMQ (AMQP)
    * Lombok (opcional, para reducir código boilerplate)
    * Nombre del Artefacto: producer-service
    * Grupo: com.eci.arcn
    * Empaquetado: Jar
    * Versión de Java: 17
3. Configura application.properties:

spring.application.name=producer-service

```
# producer-service/src/main/resources/application.properties
server.port=8080

# Configuración de RabbitMQ
# El host será 'rabbitmq' cuando se ejecute en Docker Compose
# Para pruebas locales en Codespaces (si corres RabbitMQ localmente), usa 'localhost'
spring.rabbitmq.host=rabbitmq
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

# Nombre de la cola y exchange (puedes centralizar esto)
app.rabbitmq.exchange=messages.exchange
app.rabbitmq.queue=messages.queue
app.rabbitmq.routingkey=messages.routingkey
```

4. Configura RabbitMQ (Exchange, Queue, Binding):

```java
package com.eci.arcn.producer_service.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    @Value("${app.rabbitmq.queue}")
    private String queueName;

    @Value("${app.rabbitmq.exchange}")
    private String exchangeName;

    @Value("${app.rabbitmq.routingkey}")
    private String routingKey;

    @Bean
    Queue queue() {
        // durable: true - la cola sobrevive a reinicios del broker
        return new Queue(queueName, true);
    }

    @Bean
    DirectExchange exchange() {
        // DirectExchange: Enruta mensajes basados en la routing key exacta
        return new DirectExchange(exchangeName);
    }

    @Bean
    Binding binding(Queue queue, DirectExchange exchange) {
        // Vincula la cola al exchange con la routing key especificada
        return BindingBuilder.bind(queue).to(exchange).with(routingKey);
    }
}
```

5. Crea un Controlador REST para Enviar Mensajes:

```java
package com.eci.arcn.producer_service.controller;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/messages")
public class MessageController {

    private static final Logger log = LoggerFactory.getLogger(MessageController.class);

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Value("${app.rabbitmq.exchange}")
    private String exchangeName;

    @Value("${app.rabbitmq.routingkey}")
    private String routingKey;

    @PostMapping("/send")
    public String sendMessage(@RequestParam String message) {
        log.info("Enviando mensaje: '{}' a exchange '{}' con routing key '{}'", message, exchangeName, routingKey);
        // Enviar el mensaje al exchange con la routing key definida
        rabbitTemplate.convertAndSend(exchangeName, routingKey, message);
        return "Mensaje '" + message + "' enviado!";
    }
}
```

6. Define el Dockerfile para el Productor:

```Dockerfile
# producer-service/Dockerfile
# Usa una imagen base de JRE ligera que coincida con tu JDK de compilación
FROM openjdk:17-jdk-slim

# Argumento para el JAR (se pasa durante la construcción en docker-compose)
ARG JAR_FILE=target/*.jar

# Establece el directorio de trabajo
WORKDIR /app

# Copia el JAR construido a la imagen
COPY ${JAR_FILE} app.jar

# Expone el puerto de la aplicación
EXPOSE 8080

# Comando para ejecutar la aplicación
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## Paso 3: Crear el Servicio Consumidor (Spring Boot)

1. Navega a la Carpeta Raíz en Codespaces: /workspaces/event-driven-lab.

2. Crea el Proyecto Consumidor: Similar al productor.
* Dependencias:
    * Spring for RabbitMQ (AMQP)
    * Lombok (opcional)
    * Nombre del Artefacto: consumer-service
    * Grupo: com.eci.arcn
    * Empaquetado: Jar
    * Versión de Java: 17
3. Configura application.properties:

```
spring.application.name=consumer-service

# consumer-service/src/main/resources/application.properties
# No necesita puerto de servidor a menos que expongas una API REST propia
# server.port=8081

# Configuración de RabbitMQ (igual que el productor)
spring.rabbitmq.host=rabbitmq
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest

# Nombre de la cola a escuchar (debe coincidir con la del productor)
app.rabbitmq.queue=messages.queue
```

4. Configura RabbitMQ (Solo la Cola es estrictamente necesaria para escuchar, pero declarar todo asegura idempotencia):

```java
package com.eci.arcn.consumer_service.config;

import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    @Value("${app.rabbitmq.queue}")
    private String queueName;

    // Declarar la cola aquí asegura que exista si el consumidor inicia primero.
    // Es importante que los parámetros (nombre, durabilidad, etc.) coincidan
    // con la declaración en el productor.
    @Bean
    Queue queue() {
        return new Queue(queueName, true); // Debe ser durable=true igual que en productor
    }
    // Nota: No es estrictamente necesario declarar el Exchange y Binding aquí
    // si el Productor ya lo hace, pero no hace daño y aumenta la resiliencia.
    // Si los declaras, asegúrate que los nombres y tipos coincidan.
}
```

5. Crea un Listener para Recibir Mensajes:

```java
package com.eci.arcn.consumer_service.listener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class MessageListener {

    private static final Logger log = LoggerFactory.getLogger(MessageListener.class);

    // Escucha en la cola definida en application.properties (a través de la config)
    @RabbitListener(queues = "${app.rabbitmq.queue}")
    public void receiveMessage(String message) {
        log.info("Mensaje recibido: '{}'", message);
        // Aquí puedes añadir la lógica para procesar el mensaje
        // Por ejemplo: guardar en base de datos, llamar a otra API, etc.
        System.out.println(">>> Mensaje Procesado: " + message);
    }
}
```

6. Define el Dockerfile para el Consumidor:

```Dockerfile
# consumer-service/Dockerfile
# Usa una imagen base de JRE ligera
FROM eclipse-temurin:17-jre-jammy

# Argumento para el JAR
ARG JAR_FILE=target/*.jar

# Directorio de trabajo
WORKDIR /app

# Copia el JAR
COPY ${JAR_FILE} app.jar

# No necesita exponer puerto si solo consume mensajes
# EXPOSE 8081

# Comando de ejecución
ENTRYPOINT ["java", "-jar", "app.jar"]
```
---

## Paso 4: Crear y subir imagenes a Docker Hub

Realizar los siguientes pasos para el productor y el consumidor

1. Generar paquete: ```mvn package```
2. Construir imagen: ```docker build -t <Nombre-Microservice> . ```
3. Etiquetar imagen: ```docker tag <Nombre-Microservice> <tu-usuario>/<Nombre-Microservice>```
4. Login en Docker Hub: ```docker login -u <tu-usuario>```
* Te pedira un password, para el cual debes generar un "Personal Access Token" en Docker Hub
5. Sube la imagen a Docker Hub: ```docker push <tu-usuario>/<Nombre-Microservice>```

---

## Paso 5: docker-compose

1. Navega a la Carpeta Raíz en Codespaces: Asegúrate de estar en /workspaces/event-driven-lab en la terminal de Codespaces.
2. Crea el archivo ```docker-compose.yml```
3. Agrega el siguiente contenido:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Servicio RabbitMQ
  rabbitmq:
    image: rabbitmq:management # Usa la imagen con la UI de gestión
    container_name: rabbitmq
    hostname: rabbitmq              # Nombre de host para que los servicios se conecten
    ports:
      - "5672:5672"   # Puerto AMQP
      - "15672:15672" # Puerto UI de Gestión (accesible vía Play With Docker)
    # volumes:
    #   - rabbitmq_data:/var/lib/rabbitmq/ # Persistencia (opcional para PWD)
    # environment:
      # Puedes mantener los de por defecto (guest/guest) o definirlos
      # RABBITMQ_DEFAULT_USER: user
      # RABBITMQ_DEFAULT_PASS: password
      # RABBITMQ_DEFAULT_VHOST: / # Virtual host por defecto
    networks:
      - event_network

  # Servicio Productor
  producer:
    image: <tudockerhubusername>/producer-service
    container_name: producer-service
    ports:
      - "8080:8080" # Expone el puerto del API REST del productor
    depends_on:
      - rabbitmq # Asegura que RabbitMQ inicie antes
    environment:
      # Sobrescribe la configuración de application.properties para asegurar el host correcto
      - SPRING_RABBITMQ_HOST=rabbitmq
      - SPRING_RABBITMQ_PORT=5672
      - SPRING_RABBITMQ_USERNAME=guest
      - SPRING_RABBITMQ_PASSWORD=guest

      # Opcional: Añadir perfiles de Spring, configuraciones de memoria, etc.
      # - SPRING_PROFILES_ACTIVE=docker
    networks:
      - event_network

  # Servicio Consumidor
  consumer:
    image: <tudockerhubusername>/consumer-service
    container_name: consumer-service
    depends_on:
      - rabbitmq
    environment:
      - SPRING_RABBITMQ_HOST=rabbitmq
      - SPRING_RABBITMQ_PORT=5672
    networks:
      - event_network
    # restart: unless-stopped # Reinicia si falla (útil en producción)

# Red para que los contenedores se comuniquen por nombre de servicio
networks:
  event_network:
    driver: bridge

# Volumen para persistencia de datos de RabbitMQ (opcional)
# volumes:
#   rabbitmq_data:
```

* Asegúrate de reemplazar tudockerhubusername con tu usuario.
* Guarda este archivo y súbelo a tu repositorio GitHub. (git add docker-compose.yml, git commit, git push).

---

## Paso 6: Ejecutar en Play with Docker

1. Ve a Play with Docker, inicia una sesión y añade una instancia.
2. Clona tu repositorio:

```bash
git clone https://github.com/<tu-usuario>/event-driven-lab.git
cd event-driven-lab/
```
3. Levanta los servicios:

```bash
docker-compose up -d
```

4. Probar eventos

1. Enviar mensaje desde producer

```bash
curl -X POST "http://localhost:8080/api/messages/send?message=HolaDesdePlayWithDocker"
```

* Deberías recibir una respuesta como ```Mensaje 'HolaDesdePlayWithDocker' enviado!```

2. Verifica el Consumidor: Revisa los logs del consumidor:

```bash
docker-compose logs consumer-service
```

* Deberías ver la línea ```Mensaje recibido: 'HolaDesdePlayWithDocker'``` y ```>>> Mensaje Procesado: HolaDesdePlayWithDocker```.

3. Explora RabbitMQ Management UI (Opcional):
* Haz clic en el botón azul del puerto 15672.
* Se abrirá la interfaz web de RabbitMQ.
* Inicia sesión con guest / guest (o las credenciales que hayas configurado).
* Ve a la pestaña "Queues", busca messages.queue. Podrás ver si hay mensajes en cola, el ritmo de entrada/salida, etc.