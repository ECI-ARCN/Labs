# Laboratorio: Creación de un Microservicio "Hello World" con Spring Boot, Docker y Play with Docker

## Introducción

Este laboratorio te guiará a través de los pasos necesarios para crear un microservicio básico "Hello World" utilizando Spring Boot, GitHub, GitHub Codespaces y Play with Docker.

## Paso 1: Configuración del Repositorio en GitHub

1. **Crear un Repositorio en GitHub:**
    * Ve a [GitHub](https://github.com) y accede a tu cuenta.
    * Haz clic en "New" para crear un nuevo repositorio.
    * Asigna un nombre al repositorio, por ejemplo, `hello-world-microservice`.
    * Inicializa el repositorio con un archivo README.

## Paso 2: Configuración de GitHub Codespaces

1. **Crear un Codespace:**
    * En la página principal del repositorio, haz clic en el botón "Code" y selecciona "Create codespace on main".
    * Esto abrirá un entorno de desarrollo en la nube donde podrás editar y ejecutar tu proyecto.

2.** Configurar Codespace:**
    * Crear archivo con el nombre ".devcontainer/devcontainer.json"
    * Pegar la siguiente configuración

```json
    {
        "name": "Java Dev Environment",
        "image": "mcr.microsoft.com/devcontainers/java:1-21",

        "features": {
        "ghcr.io/devcontainers/features/java:1": {
            "version": "none",
            "installMaven": "true",
            "mavenVersion": "3.8.6",
            "installGradle": "false"
            },
        "ghcr.io/devcontainers/features/docker-in-docker:1": {}
        },
        "customizations": {
        "vscode": {
            "extensions": [
                "vscjava.vscode-java-pack",
                "streetsidesoftware.code-spell-checker",
                "pivotal.vscode-spring-boot",
                "vmware.vscode-boot-dev-pack"
            ]
        }
        },
        "postCreateCommand": "mvn clean install"
    }
```

3. Asegurate de la instalación de Java y Maven:
    * java -version
    * mvn -version

## Paso 3: Configuración del Proyecto Spring Boot

1. **Generar el Proyecto Spring Boot:**
    * Verifica, que se tenga la extensión de **Spring Boot Extension Pack**
    ![alt text](/images/spring-boot-extension-pack.png)
    * Dirigite a la barra **Command Palette** Ctrl+Shift+P
    * Buscar y seleccionar **Spring Initializr: Create a Maven Project**  
    * Configura el proyecto con las siguientes opciones:
        - **Language:** Java
        - **Spring Boot:** Última versión estable
        - **Group:** com.eci.arcn
        - **Artifact:** microservice-helloworld
        - **Name:** microservice-helloworld
        - **Packaging:** Jar
        - **Java Version:** 17
    * Agrega la dependencia `Spring Web`.
    * Haz clic en "Generate" para descargar el proyecto.
    * **Nota:** una vez creado el proyecto, asegurate de mover los archivos a la raiz del workspace

## Paso 4: Implementación del Servicio "Hello World"

1. **Crear el Controlador:**
    * En GitHub Codespaces, navega a `src/main/java/com/eci/arcn/microservice-helloworld`.
    * Crea una nueva clase llamada `HelloWorldController.java` con el siguiente contenido:

```java

    package com.eci.arcn.helloworld;

    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class HelloWorldController {

        @GetMapping("/hello")
        public String hello() {
            return "Hello, World!";
        }
    }
```

2. **Configurar la Aplicación:**
    * Asegúrate de que la clase principal del proyecto esté anotada con `@SpringBootApplication`.

3. **Ejecutar la Aplicación:**
    * Utiliza el terminal integrado en Codespaces para ejecutar tu aplicación con el comando:
    
    ```bash
        mvn spring-boot:run
    ```

    * Verifica que la aplicación se ejecute correctamente y esté accesible ejecutando
    
    ```bash
        curl http://localhost:8080/hello
    ```

## Paso 5: Crear y Subir la Imagen Docker

1. **Crear el Dockerfile:**
    * En la raíz del proyecto, crea un archivo llamado `Dockerfile` con el siguiente contenido:

    ```dockerfile
        FROM openjdk:17-jdk-slim
        COPY target/microservice-helloworld.jar microservice-helloworld.jar
        ENTRYPOINT ["java", "-jar", "/microservice-helloworld.jar"]
    ```

2. **Construir la Imagen Docker:**
    * Ejecuta el siguiente comando para compilar el proyecto y generar el archivo JAR:
    ```bash
        mvn clean package
    ```
    * Construye la imagen Docker con el comando:
    ```bash
        docker build -t microservice-helloworld .
    ```

3. **Subir la Imagen a Docker Hub:**
    * Asegurate de tener/crear cuenta en [Docker Hub](https://hub.docker.com/)
    * Etiqueta la imagen con tu nombre de usuario de Docker Hub:
    
    ```bash
        docker tag microservice-helloworld <tu-usuario>/microservice-helloworld
    ```

    * Inicia sesión en Docker Hub:
        - Github codespace, genera un login del space a Docker Hub, por tanto debes desloguear primero
        
    ```bash
        docker logout
    ```

    * Una vez deslogueado, loguearse con el usuario de Docker Hub
        
    ```bash
        docker login -u <tu-usuario>
    ```

    * Te pedira un password, para el cual debes generar un "Personal Access Token" en Docker Hub
    * Sube la imagen a Docker Hub:
     
    ```bash
        docker push <tu-usuario>/microservice-helloworld
    ```

## Paso 6: Ejecutar el Servicio en Killercoda

> **¿Por qué Killercoda?** Es una plataforma gratuita que provee entornos de Linux con Docker preinstalado, accesibles desde el navegador sin necesidad de instalar nada adicional. Es más estable y confiable que Play with Docker.

### Opción A: Usar Killercoda (entorno online con URL pública)

1. **Acceder a Killercoda:**
    * Ve a [Killercoda Ubuntu Playground](https://killercoda.com/playgrounds/scenario/ubuntu) e inicia sesión (puedes usar tu cuenta de GitHub o Google).
    * Verás un terminal de Linux listo para usar con Docker preinstalado.

2. **Verificar Docker:**
    * En el terminal de Killercoda, verifica que Docker está disponible:
    ```bash
        docker --version
    ```

3. **Ejecutar el Contenedor:**
    * Ejecuta tu imagen publicada en Docker Hub:
    ```bash
        docker run -d -p 8080:8080 <tu-usuario>/microservice-helloworld
    ```
    * El flag `-d` ejecuta el contenedor en segundo plano (detached mode).

4. **Verificar que el servicio está activo:**

    > ⏳ **Espera ~20 segundos** antes de ejecutar el `curl`. Spring Boot necesita tiempo para iniciar. Puedes verificar que está listo con:
    > ```bash
    >     docker logs $(docker ps -lq)
    > ```
    > Busca el mensaje `Started MicroserviceHelloworldApplication in X seconds` para confirmar que está listo.

    > ⚠️ **Si el curl no devuelve respuesta o devuelve un error 401:** Verifica que tu `pom.xml` **no incluya** la dependencia `spring-boot-starter-security`. Si está presente, elimínala y reconstruye la imagen. Spring Security protege todos los endpoints por defecto y bloqueará el acceso a `/hello`.

    ```bash
        curl http://localhost:8080/hello
    ```
    * Deberías ver la respuesta: `Hello, World!`

5. **Acceder al servicio desde el navegador:**
    * Haz clic en el botón **"Traffic / Port"** ubicado en la esquina superior derecha del terminal de Killercoda.
    * Ingresa el puerto `8080` y confirma.
    * Killercoda generará una **URL pública temporal**. Añade `/hello` al final para ver el mensaje `Hello, World!` en el navegador.

    > **Nota:** La URL pública de Killercoda es válida solo durante tu sesión activa (máximo **1 hora** en el plan gratuito). Asegúrate de haber completado los Pasos 1–5 con anticipación.

---

### Opción B: Verificar directamente en GitHub Codespaces (sin plataforma externa)

Si ya tienes tu imagen publicada en Docker Hub, puedes ejecutar y verificar el microservicio **directamente en el mismo Codespace** que usaste en los pasos anteriores, sin necesidad de ninguna plataforma adicional.

1. **Ejecutar el contenedor en Codespaces:**
    ```bash
        docker run -d -p 8080:8080 <tu-usuario>/microservice-helloworld
    ```

2. **Verificar la respuesta del servicio:**
    ```bash
        curl http://localhost:8080/hello
    ```
    * Deberías ver: `Hello, World!`

3. **Acceder desde el navegador (opcional):**
    * GitHub Codespaces detectará automáticamente el puerto `8080` abierto y mostrará una notificación en la esquina inferior derecha.
    * Haz clic en **"Open in Browser"** o ve a la pestaña **"Ports"** del panel inferior de VS Code y abre el puerto `8080`.
    * Añade `/hello` a la URL que se genera para ver el mensaje en el navegador.