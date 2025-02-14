# Laboratorio de Principios SOLID en Java

Este laboratorio tiene como objetivo que los estudiantes refactoricen código que viola los principios SOLID y apliquen las mejores prácticas.

## Requisitos Previos
- Java 17+
- Maven
- GitHub Codespaces
- JUnit 5 para pruebas

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
Cada estudiante debe crear una versión refactorizada en `GoodExample.java` dentro de cada carpeta (`srp/`, `ocp/`, etc.), aplicando SOLID correctamente.