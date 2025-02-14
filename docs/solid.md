# Laboratorio de Principios SOLID en Java

Este laboratorio tiene como objetivo que los estudiantes refactoricen código que viola los principios SOLID y apliquen las mejores prácticas.

## Requisitos Previos

- Java 17+
- Maven
- GitHub Codespaces
- JUnit 5 para pruebas

## Creación del Repositorio en GitHub

1. **Crear un nuevo repositorio:**
   - Ve a [GitHub](https://github.com/).
   - Haz clic en el botón "New repository".
   - Asigna un nombre, por ejemplo, `solid-principles-java-lab`.
   - Selecciona "Public" o "Private" según prefieras.
   - Inicializa el repositorio con un archivo `README.md`.
   - Haz clic en "Create repository".

## Creación del Proyecto Maven

Para desarrollar el laboratorio, primero debemos crear un proyecto Maven. En GitHub Codespaces, ejecuta el siguiente comando en la terminal:

```sh
mvn archetype:generate -DgroupId=com.example.solid -DartifactId=solid-principles-java-lab -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

Este comando generará la estructura básica del proyecto con las carpetas `src/main/java` y `src/test/java`.

Luego, navega al directorio del proyecto:

```sh
cd solid-principles-java-lab
```

Ahora puedes continuar con la configuración del entorno.

## Configuración del POM.xml

Para asegurar la compatibilidad con Java 17, actualiza el archivo `pom.xml` con la siguiente configuración:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.solid</groupId>
    <artifactId>solid-principles-java-lab</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>5.7.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.7.1</version>
        </dependency>
    </dependencies>
</project>
```

Ahora puedes continuar con la configuración del entorno.

## Configuración en GitHub Codespaces

Para desarrollar el laboratorio en GitHub Codespaces, sigue estos pasos:

1. **Habilitar Codespaces en el repositorio:**

   - Ve a tu repositorio en GitHub.
   - Haz clic en el botón "Code" y selecciona "Codespaces".
   - Crea un nuevo Codespace.

2. **Configurar el entorno de desarrollo:**

   - Asegúrate de que `Java 17` y `Maven` estén instalados.
   - Puedes definir un archivo `.devcontainer/devcontainer.json` para personalizar el entorno:

```json
{
  "image": "mcr.microsoft.com/devcontainers/java:17",
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "17"
    }
  },
  "postCreateCommand": "sudo apt update && sudo apt install -y maven && mvn clean install"
}
```

3. **Compilar y ejecutar pruebas:**
   - Usa el siguiente comando para compilar el código y ejecutar pruebas:
     ```sh
     mvn test
     ```

## Estructura del Proyecto

```
solid-principles-java-lab/
├── src/
│   ├── main/java/com/example/solid/
│   │   ├── srp/  (Single Responsibility Principle)
│   │   ├── ocp/  (Open/Closed Principle)
│   │   ├── lsp/  (Liskov Substitution Principle)
│   │   ├── isp/  (Interface Segregation Principle)
│   │   ├── dip/  (Dependency Inversion Principle)
│   ├── test/java/com/example/solid/  (Pruebas unitarias)
├── pom.xml
├── README.md
```

## Ejercicios

### 1. Single Responsibility Principle (SRP)

**Problema:** La clase `Invoice` tiene múltiples responsabilidades.

```java
public class Invoice {
    private String customer;
    private double amount;

    public Invoice(String customer, double amount) {
        this.customer = customer;
        this.amount = amount;
    }

    public double calculateTotal() {
        return amount * 1.21;
    }

    public void printInvoice() {
        System.out.println("Factura para: " + customer);
        System.out.println("Total: " + calculateTotal());
    }

    public void saveToDatabase() {
        System.out.println("Guardando factura...");
    }
}
```

**Tarea:** Refactoriza separando la lógica en diferentes clases.

---

### 2. Open/Closed Principle (OCP)

**Problema:** La clase `DiscountCalculator` no es extensible sin modificar su código.

```java
public class DiscountCalculator {
    public double calculateDiscount(String customerType, double price) {
        if (customerType.equals("Regular")) {
            return price * 0.10;
        } else if (customerType.equals("VIP")) {
            return price * 0.20;
        }
        return 0;
    }
}
```

**Tarea:** Aplica OCP creando una interfaz `DiscountStrategy`.

---

### 3. Liskov Substitution Principle (LSP)

**Problema:** `ElectricCar` no puede usar `refuel()`.

```java
public class Car {
    public void drive() {
        System.out.println("Conduciendo...");
    }

    public void refuel() {
        System.out.println("Repostando...");
    }
}

class ElectricCar extends Car {
    @Override
    public void refuel() {
        throw new UnsupportedOperationException("Los coches eléctricos no usan combustible.");
    }
}
```

**Tarea:** Separa los métodos en interfaces específicas.

---

### 4. Interface Segregation Principle (ISP)

**Problema:** La interfaz `Worker` obliga a los desarrolladores a implementar métodos innecesarios.

```java
public interface Worker {
    void work();
    void eat();
}

class Developer implements Worker {
    @Override
    public void work() {
        System.out.println("Escribiendo código...");
    }

    @Override
    public void eat() {
        throw new UnsupportedOperationException("Sin horario fijo de almuerzo.");
    }
}
```

**Tarea:** Divide la interfaz en `Workable` y `Eatable`.

---

### 5. Dependency Inversion Principle (DIP)

**Problema:** `OrderProcessor` depende directamente de `MySQLDatabase`.

```java
public class MySQLDatabase {
    public void saveOrder() {
        System.out.println("Guardando pedido en MySQL...");
    }
}

class OrderProcessor {
    private MySQLDatabase database;

    public OrderProcessor() {
        this.database = new MySQLDatabase();
    }

    public void processOrder() {
        System.out.println("Procesando pedido...");
        database.saveOrder();
    }
}
```

**Tarea:** Crea una interfaz `Database` e inyecta una implementación concreta.

---

### ¿Cómo entregar las soluciones?

1. Cada estudiante/grupo debe crear una versión refactorizada en `GoodExample.java` dentro de cada carpeta (`srp/`, `ocp/`, etc.), aplicando SOLID correctamente.
2. Se debe realizar UT en cada caso y/o principio.
3. Generar un Readme en el que se indique nombres de los integrantes del grupo y patrones de diseño que se pueden implementar en los retos.
4. Se debe contemplar el trabajo colaborativo (Todos deben tener un cambio con su usuario en el repositorio).
5. Se debe establecer una estrategia para exponer el ejercicio en el que cada integrante participe. 

